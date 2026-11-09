# Chapter 16: The PlaywrightWorld Deep Dive

---

Every Cucumber scenario starts with a fresh World object. This World carries the state that steps share during a single scenario -- the browser, the page, the URL, the wait times, and any custom data you need. In webship-js, that World is `PlaywrightWorld`, and understanding it is the key to writing powerful custom steps.

This chapter takes you inside PlaywrightWorld: what it does, how it works, and how to extend it for your project's specific needs.

---

## What Is the World Object?

In Cucumber, the World is the `this` context inside every step definition. When you write:

```javascript
Given('I am on homepage', async function () {
  await this.page.goto(this.launchUrl);
});
```

The `this` is the World object. Every step in a single scenario shares the same World instance, which means they share the same `this.page`, `this.launchUrl`, and any other properties.

```
┌──────────────────────────────────────────────────────────────────┐
│ World Lifecycle per Scenario                                     │
│                                                                  │
│  Scenario: Login Test                                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ new PlaywrightWorld()  ← Fresh World created                │ │
│  │                                                             │ │
│  │ Before Hook:                                                │ │
│  │   this.openBrowser()   ← Browser launched                   │ │
│  │   this.page created    ← New page/tab opened                │ │
│  │                                                             │ │
│  │ Step 1: Given I am on homepage                              │ │
│  │   this.page.goto(this.launchUrl)                            │ │
│  │                                                             │ │
│  │ Step 2: When I fill in "Email" with "admin@e.c"             │ │
│  │   this.page.getByLabel("Email").fill("admin@e.c")           │ │
│  │                                                             │ │
│  │ Step 3: And I press "Sign In"                               │ │
│  │   this.page.getByRole("button", {name: "Sign In"}).click()  │ │
│  │                                                             │ │
│  │ Step 4: Then I should see "Dashboard"                       │ │
│  │   this.page.locator("body").filter({hasText: "Dashboard"})  │ │
│  │                                                             │ │
│  │ After Hook:                                                 │ │
│  │   this.closeBrowser()  ← Browser closed                     │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Scenario: Signup Test                                           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ new PlaywrightWorld()  ← COMPLETELY NEW World               │ │
│  │                        (no state from Login Test)           │ │
│  │ ...                                                         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Critical concept:** Each scenario gets a **new** World instance. No state carries over between scenarios. This is how Cucumber ensures test isolation.

It is also a useful debugging lens. When a scenario passes alone but fails in the full suite, the leak is almost never in the World -- it *cannot* be, because the World is rebuilt every time. The state is leaking through something outside it: the application database, a shared user account, a file on disk. Knowing what the World guarantees tells you where not to waste your afternoon looking.

---

## PlaywrightWorld Internals

<!-- diagram: playwright-world -->

Here is the actual PlaywrightWorld class from webship-js:

```javascript
class PlaywrightWorld extends World {
  constructor(options) {
    super(options);
    this.launchUrl = this.parameters.launchUrl;
    this.minWaitTime = this.parameters.minWaitTime;
    this.playwrightBrowser = null;
    this.context = null;
    this.page = null;
    this.assetsFolder = path.join(__dirname, '../assets/');
  }

  async openBrowser() {
    const { browser: browserName, launchOptions, contextOptions } = playwrightConfig;
    this.playwrightBrowser = await playwright[browserName].launch(launchOptions);
    this.context = await this.playwrightBrowser.newContext(contextOptions);
    this.page = await this.context.newPage();
  }

  async closeBrowser() {
    if (this.playwrightBrowser) {
      await this.playwrightBrowser.close();
      this.playwrightBrowser = null;
      this.context = null;
      this.page = null;
    }
  }
}
```

Where do the constructor values come from? `this.parameters` is the worldParameters block from cucumber.js. In webship-js 2.0 the stock profile sets launchUrl from the LAUNCH_URL environment variable (default http://localhost:8080), minWaitTime.page to 3000ms, and a selectors registry: named CSS and XPath selectors auto-loaded from JSON files in tests/selectors/, plus the breakpoints table and the 60px scroll offset the selector steps use. openBrowser() reads playwright.config.ts, which is itself environment-driven -- BROWSER picks chromium, firefox, or webkit; HEADLESS='false' switches to headed; SLOW_MO defaults to 0 in CI, 300 headless, 800 headed; viewport is null and ignoreHTTPSErrors is true. Nothing about the World is hardcoded, which is exactly what lets one suite run unmodified against five environments.

The context created in openBrowser() is also where webship-js installs its smart-wait instrumentation. An addInitScript hook injects three window globals into every page before any application code runs: window.__webshipAjaxCount (a fetch/XHR wrapper counting in-flight requests), window.__webshipPendingTimers (a setTimeout wrapper counting pending timers), and window.__webshipLastMutation (a MutationObserver timestamp). Every built-in step's auto-settle reads those globals through this.page. Remember that when you reach Recipe 160: replace the World and skip the stock browser lifecycle, and the entire smart-wait system silently disappears with it.

**Properties breakdown:**

```
┌──────────────────────────────────────────────────────────────────┐
│ PlaywrightWorld Properties                                       │
│                                                                  │
│ Property          │ Source           │ Purpose                    │
│ ──────────────────┼─────────────────┼──────────────────────────  │
│ this.launchUrl    │ cucumber.js      │ Base URL for navigation   │
│                   │ worldParameters  │ e.g., http://localhost:80 │
│                   │                  │                            │
│ this.minWaitTime  │ cucumber.js      │ Timing delays (object):   │
│                   │ worldParameters  │  .before_scenario         │
│                   │                  │  .after_scenario          │
│                   │                  │  .before_step             │
│                   │                  │  .after_step              │
│                   │                  │  .page (page load wait)   │
│                   │                  │                            │
│ this.playwright   │ openBrowser()    │ Playwright browser        │
│ Browser           │                  │ instance (chromium/ff/wk) │
│                   │                  │                            │
│ this.context      │ openBrowser()    │ Browser context (like an  │
│                   │                  │ incognito window)          │
│                   │                  │                            │
│ this.page         │ openBrowser()    │ The page/tab you interact │
│                   │                  │ with in step definitions   │
│                   │                  │                            │
│ this.assetsFolder │ constructor      │ Path to test asset files  │
│                   │                  │ (upload files, etc.)       │
└──────────────────────────────────────────────────────────────────┘
```

**The Playwright hierarchy:**

```
Browser (one per scenario)
  └── Context (like an incognito session)
       └── Page (the tab where tests run)
            ├── page.goto()       ← navigation
            ├── page.locator()    ← find elements
            ├── page.getByLabel() ← find by label
            ├── page.keyboard     ← key presses
            ├── page.mouse        ← mouse actions
            └── page.screenshot() ← capture screen
```

---

## Helper Functions

webship-js defines several helper functions that you can reuse in your custom step definitions:

### buildSelector()

```javascript
function buildSelector(attrValue, attr) {
  // "#id" or ".class" → return as-is
  // "name" (no spaces) → try tag, #id, .class, [name=], [value=], [placeholder=], [title=]
  // "two words" → try [value=], [placeholder=]
  // with attr arg → [attr="value" i] (case insensitive)
}
```

```
buildSelector Examples:
┌────────────────────────────────────────────────────────────────┐
│ Input                     │ Output CSS Selector                │
├────────────────────────────────────────────────────────────────┤
│ buildSelector("#myId")    │ #myId                              │
│ buildSelector(".myClass") │ .myClass                           │
│ buildSelector("email")    │ email,#email,.email,[name=email],  │
│                           │ [value="email"],                    │
│                           │ [placeholder="email"],              │
│                           │ [title="email"]                    │
│ buildSelector("my text")  │ [value="my text"],                 │
│                           │ [placeholder="my text"]             │
│ buildSelector("x","data-id")│ [data-id="x" i]                 │
└────────────────────────────────────────────────────────────────┘
```

### fillField()

```javascript
async function fillField(page, field, value) {
  // 1. Try: page.getByLabel(field, { exact: true })
  // 2. Try: page.getByPlaceholder(field, { exact: true })
  // 3. Fallback: page.locator(`[name="${field}"]`)
}
```

This is the label-first resolution behind the built-in `When I fill in "Label" with "value"` step. It is also a quiet design statement: tests written against visible labels survive markup refactors that would break CSS selectors, and they fail loudly when a label disappears -- which is a real usability bug, not a test maintenance chore.

### gotoUrl()

```javascript
async function gotoUrl(page, url) {
  // Navigates to URL with waitUntil: 'domcontentloaded'
  // Silently handles NS_ERROR_NET_EMPTY_RESPONSE (Firefox quirk)
}
```

---

## Recipe 159: Adding Custom Properties to the World

**Problem:** Your scenario creates a product in step two and needs to assert on that exact product in step six. Hardcoding the name in both steps works -- until the third time a new hire copies the feature file, changes one of the two hardcoded strings, and your team spends an afternoon debugging a "flaky" test that was never flaky. Steps need a way to hand data to later steps without anyone keeping two strings in sync by hand.

**Solution:**

```javascript
// tests/step-definitions/custom-world-properties.js
const { Given, When, Then } = require('@cucumber/cucumber');

Given('I create a test product named {string}', async function (name) {
  // Store the product name on the World for later steps
  this.testProductName = name;

  await this.page.goto(this.launchUrl + '/admin/products/new');
  await this.page.getByLabel('Name').fill(name);
  await this.page.getByRole('button', { name: 'Save' }).click();
  await this.page.waitForLoadState('networkidle');
});

Then('I should see the test product in the list', async function () {
  // Access the stored name from a previous step
  const locator = this.page.locator('body').filter({ hasText: this.testProductName });
  await locator.waitFor({ timeout: 5000 });
});
```

```gherkin
Feature: Product Management
  Scenario: Create and verify product
    Given I am a logged in user with the "admin" user
    And I create a test product named "Test Widget"
    When I go to "/admin/products"
    And wait for AJAX to finish
    Then I should see the test product in the list
```

**How It Works:**

Since all steps in a scenario share the same World (`this`), a property set in one step is readable in every later step -- no globals, no module-level variables, no files on disk. Cleanup is automatic and absolute: the next scenario constructs a brand-new PlaywrightWorld, so `this.testProductName` cannot leak forward even if the scenario fails halfway through. That guarantee is the reason to prefer World properties over the tempting alternative -- a module-level `let lastProduct` in a step file survives across scenarios and quietly turns scenario *order* into hidden test logic; `this` never does.

One naming habit pays off: prefix your custom properties (`this.testProductName`, not `this.name`) so they cannot collide with what the framework already owns. page, context, launchUrl, and minWaitTime are taken, and overwriting this.page mid-scenario produces spectacular confusion.

**At scale:** Agree on the property names once and write them down -- this.testData for created records, this.lastApiResponse for API payloads -- in the shared step library every project consumes. When the same QA engineer rotates across six projects in a quarter, finding the same state in the same place each time is the difference between contributing on day one and reverse-engineering each suite's private conventions.

---

## Recipe 160: Extending PlaywrightWorld

**Problem:** The same fifteen lines of login automation appear in four step files, each slightly diverged -- one waits for networkidle, one does not, one still uses last quarter's password. When the login flow changes, you fix three of the four, and the suite goes intermittently red in a way nobody can explain for a week. Logic that several steps share belongs on the World as a method: written once, called everywhere.

**Solution:**

```javascript
// tests/support/extended-world.js
const { setWorldConstructor, World } = require('@cucumber/cucumber');
const playwright = require('playwright');
const playwrightConfig = require(require('path').join(process.cwd(), 'playwright.config'));
const path = require('path');

class ExtendedWorld extends World {
  constructor(options) {
    super(options);
    this.launchUrl = this.parameters.launchUrl;
    this.minWaitTime = this.parameters.minWaitTime;
    this.playwrightBrowser = null;
    this.context = null;
    this.page = null;
    this.assetsFolder = path.join(__dirname, '../assets/');
  }

  async openBrowser() {
    const { browser: browserName, launchOptions, contextOptions } = playwrightConfig;
    this.playwrightBrowser = await playwright[browserName].launch(launchOptions);
    this.context = await this.playwrightBrowser.newContext(contextOptions);
    this.page = await this.context.newPage();
  }

  async closeBrowser() {
    if (this.playwrightBrowser) {
      await this.playwrightBrowser.close();
      this.playwrightBrowser = null;
    }
  }

  // Custom method: login programmatically
  async loginAs(role) {
    const users = {
      admin: { email: 'admin@example.com', password: 'AdminPass123' },
      editor: { email: 'editor@example.com', password: 'EditorPass123' },
    };
    const user = users[role];
    await this.page.goto(this.launchUrl + '/login');
    await this.page.getByLabel('Email').fill(user.email);
    await this.page.getByLabel('Password').fill(user.password);
    await this.page.getByRole('button', { name: 'Sign In' }).click();
    await this.page.waitForLoadState('networkidle');
  }

  // Custom method: wait for and dismiss any toast notification
  async dismissToast() {
    const toast = this.page.locator('.toast, .snackbar, .notification').first();
    if (await toast.isVisible()) {
      const closeBtn = toast.locator('.close, [aria-label="Close"]').first();
      if (await closeBtn.isVisible()) await closeBtn.click();
    }
  }

  // Custom method: take a named screenshot
  async screenshot(name) {
    const dir = path.join(process.cwd(), 'screenshots');
    await this.page.screenshot({
      path: path.join(dir, `${name}.png`),
      fullPage: true,
    });
  }
}

setWorldConstructor(ExtendedWorld);
```

Then use the methods in step definitions:

```javascript
const { Given, When } = require('@cucumber/cucumber');

Given('I am logged in as {string}', async function (role) {
  await this.loginAs(role);
});

When('I take a screenshot named {string}', async function (name) {
  await this.screenshot(name);
});
```

**How It Works:**

`setWorldConstructor(ExtendedWorld)` tells Cucumber to instantiate your class for every scenario -- and there can be only one World constructor per suite, so this *replaces* PlaywrightWorld rather than wrapping it. That is why the example re-implements openBrowser() and closeBrowser() with the same shape: webship-js's Before and After hooks call those methods by name, and all 411 built-in steps expect this.page, this.context, and this.launchUrl to exist exactly as the stock World provides them. Keep the constructor and browser lifecycle identical and layer your methods on top; change the lifecycle and you will discover that the smart-wait instrumentation, the screenshot hooks, and every built-in step assumed the contract you just broke -- and none of them fail with a message that says so.

The methods themselves follow a rule worth adopting: World methods for *operations* (loginAs, dismissToast), step definitions for *language*. Steps stay one line each, the operation is reusable from any step file, and a Gherkin reader never sees the plumbing.

For login specifically, check the built-in option before writing loginAs() everywhere: webship-js ships storage-state steps (auth.steps.js) that save a logged-in session once and restore it per scenario -- the saved storageState carries cookies and localStorage, so restoring it skips the login UI entirely. loginAs() earns its place when you genuinely need to exercise the login form, or when roles switch mid-feature.

**At scale:** The ExtendedWorld is the single highest-leverage file to share across projects. Publish it, with your shared steps, as an internal npm package -- @yourorg/test-world -- and each project's tests/support/ becomes a two-line import. When the design system renames every toast selector, you fix dismissToast() once, version-bump, and twenty suites pick it up through a dependency update on the next pipeline run -- instead of twenty near-identical merge requests reviewed by twenty different people, three of which quietly diverge.

---

## Recipe 161: Multiple World Configurations

**Problem:** The suite passes against localhost and fails against staging, and the "fix" on someone's laptop was editing launchUrl in cucumber.js -- which then got committed and broke everyone else's local runs. The Tuesday before release, the suite must point at staging; Wednesday morning, at production smoke checks. Nobody should be editing config files to make that happen.

**Solution:**

```javascript
// cucumber.js
const common = {
  require: ['tests/step-definitions/**/*.js'],
  format: ['@cucumber/pretty-formatter'],
};

module.exports = {
  default: {
    ...common,
    worldParameters: {
      launchUrl: 'http://localhost:8080',
      minWaitTime: {
        before_scenario: 0, after_scenario: 0,
        before_step: 0, after_step: 0, page: 3000,
      },
    },
  },

  staging: {
    ...common,
    worldParameters: {
      launchUrl: 'https://staging.example.com',
      minWaitTime: {
        before_scenario: 1000, after_scenario: 500,
        before_step: 200, after_step: 200, page: 5000,
      },
    },
  },

  production: {
    ...common,
    worldParameters: {
      launchUrl: 'https://www.example.com',
      minWaitTime: {
        before_scenario: 1000, after_scenario: 500,
        before_step: 300, after_step: 300, page: 8000,
      },
    },
  },
};
```

Run with a specific profile:

```bash
# Local (default)
npx cucumber-js

# Staging
npx cucumber-js --profile staging

# Production
npx cucumber-js --profile production
```

**How It Works:**

A Cucumber profile is just a named bundle of options. `--profile staging` swaps the entire worldParameters block without touching code, and every scenario's World constructor receives the staging values through this.parameters. The slower minWaitTime values on the remote profiles reflect a real difference -- staging usually runs on shared infrastructure behind a CDN, and a page that settles in one second locally can take five. But notice what does *not* change: wait budgets are configuration, never feature-file content, so the same Gherkin runs against all three environments unmodified. That is the whole game.

Before reaching for profiles, know that webship-js's stock cucumber.js already reads LAUNCH_URL from the environment (default http://localhost:8080) -- `LAUNCH_URL=https://staging.example.com npx cucumber-js` covers the most common case with zero config files. Profiles earn their place when environments differ in more than the URL: wait budgets, selector files, report settings.

**At scale:** In CI, prefer the env var over the profile. Pipelines already manage per-environment variables and secrets, so injecting LAUNCH_URL from the pipeline environment means one pipeline template serves all 20 projects, and adding a new environment is a CI variable -- not a pull request to every repo. Keep profiles for the human-shaped differences (a slow-staging wait budget a developer wants locally), let the pipeline own the URLs, and never let production URLs or credentials live in a committed config file.

---

## Chapter Summary

- The World object is `this` in every step definition -- shared within a scenario, fresh between scenarios
- PlaywrightWorld extends Cucumber's World with browser, context, page, and configuration
- Browser lifecycle: Before hook opens browser → steps run → After hook closes browser
- Helper functions (buildSelector, fillField, gotoUrl) simplify element interaction
- Store custom state on `this` to share data between steps in a scenario
- Extend PlaywrightWorld with custom methods for project-specific operations
- Use Cucumber profiles for different environments (local, staging, production)
- Each scenario gets complete isolation: new World, new browser, new context

Next: Writing Custom Step Definitions -- creating your own Gherkin vocabulary.

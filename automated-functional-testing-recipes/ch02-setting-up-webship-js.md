# Chapter 2: Setting Up webship-js

---

## 2.1 What Is webship-js?

webship-js is an automated functional acceptance testing tool. Let me break down every word in that description because each one matters:

- **Automated**: Tests run themselves. No human clicks through the application.
- **Functional**: Tests verify that features work correctly. Not performance, not security -- functional correctness.
- **Acceptance**: Tests are written from the user's perspective. They verify what the user accepts as "working."
- **Testing tool**: It is a framework, not a service. It runs on your machine or CI server.

Under the hood, webship-js combines three technologies:

1. **Cucumber-js** -- the JavaScript implementation of Cucumber. Parses Gherkin feature files and matches steps to JavaScript functions.

2. **Playwright** -- Microsoft's browser automation library (version 1.58 in webship-js 2.0). Controls Chromium, Firefox, and WebKit browsers. Handles navigation, clicking, typing, waiting, screenshots, and more.

3. **Pre-built step definitions** -- 411 built-in steps organized into 36 step files, covering navigation, form interactions, web-first assertions, waiting, scrolling, modals and dialogs, cookies, accessibility audits, network mocking, video recording, and API testing. Each step file owns one domain (`form.steps.js`, `api.steps.js`, `a11y.steps.js`, and so on), which is also how you will find things when you want to read what a step really does.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR TEST SUITE                          │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Feature Files (.feature)                                 │  │
│  │                                                           │  │
│  │  Feature: Login                                           │  │
│  │    Scenario: Valid credentials                            │  │
│  │      Given I am on homepage                               │  │
│  │      When I go to "/login"                                │  │
│  │      And I fill in "Email" with "user@test.com"           │  │
│  │      And I press "Sign In"                                │  │
│  │      Then I should see "Dashboard"                        │  │
│  │                                                           │  │
│  │  YOU WRITE THESE                                          │  │
│  └─────────────────────────────┬─────────────────────────────┘  │
│                                │                                │
│                                │ Cucumber-js matches steps      │
│                                ▼                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Step Definitions (webship-js provides these)             │  │
│  │                                                           │  │
│  │  Given('I am on homepage', async function() {             │  │
│  │    await this.page.goto(this.launchUrl);                  │  │
│  │  });                                                      │  │
│  │                                                           │  │
│  │  When('I fill in {string} with {string}',                 │  │
│  │    async function(field, value) {                         │  │
│  │      // Playwright fills the field                        │  │
│  │    });                                                    │  │
│  │                                                           │  │
│  │  WEBSHIP-JS PROVIDES THESE (411 built-in)                │  │
│  └─────────────────────────────┬─────────────────────────────┘  │
│                                │                                │
│                                │ Step definitions use           │
│                                ▼                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  PlaywrightWorld (Custom Cucumber World)                  │  │
│  │                                                           │  │
│  │  ┌──────────────────────────────────────────────┐        │  │
│  │  │  this.browser    (Playwright browser)        │        │  │
│  │  │  this.context    (browser context)           │        │  │
│  │  │  this.page       (active page/tab)           │        │  │
│  │  │  this.launchUrl  (base URL)                  │        │  │
│  │  │  this.helpers    (buildSelector, etc)        │        │  │
│  │  └──────────────────────────────────────────────┘        │  │
│  │                                                           │  │
│  │  FRESH INSTANCE PER SCENARIO (isolation guaranteed)       │  │
│  └─────────────────────────────┬─────────────────────────────┘  │
│                                │                                │
│                                │ Playwright drives              │
│                                ▼                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Playwright (Browser Automation Engine)                   │  │
│  │                                                           │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐           │  │
│  │  │ Chromium  │  │  Firefox  │  │  WebKit   │           │  │
│  │  │ (Chrome)  │  │           │  │  (Safari) │           │  │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘           │  │
│  │        │              │              │                   │  │
│  └────────┼──────────────┼──────────────┼───────────────────┘  │
│           │              │              │                       │
└───────────┼──────────────┼──────────────┼───────────────────────┘
            │              │              │
            ▼              ▼              ▼
     ┌────────────────────────────────────────┐
     │                                        │
     │     YOUR WEB APPLICATION               │
     │     (System Under Test)                │
     │                                        │
     │     http://localhost:8080              │
     │     https://staging.example.com       │
     │     https://www.example.com           │
     │                                        │
     └────────────────────────────────────────┘
```

### How It Compares

You might be wondering how webship-js compares to other testing tools. Here is an honest comparison:

```
┌──────────────────┬────────────┬────────────┬────────────┬────────────┐
│                  │ webship-js │  Cypress   │ Selenium   │ Playwright │
│                  │            │            │ + Cucumber │ (raw)      │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Language         │ Gherkin +  │ JavaScript │ Multiple   │ JavaScript │
│                  │ JavaScript │            │            │            │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ BDD Native?      │ Yes        │ No (addon) │ Yes        │ No         │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Browser Support  │ Chromium   │ Chromium   │ All        │ Chromium   │
│                  │ Firefox    │ Firefox    │            │ Firefox    │
│                  │ WebKit     │ (limited)  │            │ WebKit     │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Pre-built Steps  │ 411        │ None       │ None       │ None       │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Setup Time       │ Minutes    │ Minutes    │ Hours      │ Minutes    │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Custom JS Needed │ Minimal    │ Everything │ Moderate   │ Everything │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Non-dev Friendly │ Very       │ No         │ Somewhat   │ No         │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ API Testing      │ Built-in   │ Built-in   │ Addon      │ Manual     │
├──────────────────┼────────────┼────────────┼────────────┼────────────┤
│ Auto-wait        │ Yes        │ Yes        │ Manual     │ Yes        │
└──────────────────┴────────────┴────────────┴────────────┴────────────┘
```

The key advantage of webship-js is **time to first test**. With Cypress or raw Playwright, you need to write all your test logic in JavaScript. With webship-js, you write Gherkin and the step definitions are already there.

**At scale:** Time to first test compounds across a portfolio. With a JavaScript-first tool, every new project starts with a developer building helpers, waits, and page objects before QA writes test one -- and every project's helpers end up slightly different, so nothing transfers. With webship-js, the 411 steps are the same on project one and project forty. A QA engineer who can test your flagship product can open any other repo in your organization and be writing scenarios within the hour. That portability is the difference between a testing tool and a testing standard.

---

## 2.2 Prerequisites and Installation

### System Requirements

- **Node.js**: Version 20 or higher (LTS recommended)
- **npm**: Version 9 or higher (comes with Node.js)
- **Operating System**: Linux, macOS, or Windows
- **Disk Space**: ~500MB (for Playwright browsers)
- **RAM**: 4GB minimum (8GB recommended for parallel testing)

### Checking Your Environment

Open your terminal and verify:

```bash
# Check Node.js version
node --version
# Expected: v20.x.x or higher

# Check npm version
npm --version
# Expected: 9.x.x or higher
```

If Node.js is not installed or is too old, download the latest LTS from nodejs.org.

### Setup 1: Setting Up a New webship-js Project from Scratch

**Problem:** A new product just landed on your team's plate and it has zero automated tests. Manual QA is keeping it alive, the developers "plan to add tests later," and you have been asked to fix that before the next release. You need a working test project today -- not a framework evaluation, not a two-week tooling spike.

**Solution:**

```bash
# Step 1: Create the project directory
mkdir my-app-tests
cd my-app-tests

# Step 2: Initialize npm
npm init -y

# Step 3: Install webship-js
npm install webship-js --save-dev

# Step 4: Install Playwright browsers
npx playwright install

# Step 5: Create directory structure
mkdir -p tests/features
mkdir -p tests/step-definitions
mkdir -p tests/assets
```

**How It Works:**

`npm install webship-js --save-dev` installs:
- webship-js core (step definitions, PlaywrightWorld, helpers)
- @cucumber/cucumber (Gherkin parser, test runner)
- playwright (browser automation)
- axios (for API testing steps)

The `--save-dev` flag adds it to your devDependencies, which is correct because testing tools are not needed in production.

`npx playwright install` downloads the browser binaries:
- Chromium (Chrome-based browser)
- Firefox
- WebKit (Safari-based browser)

These are stored in your home directory's cache, not in the project. They are shared across all Playwright projects on your machine.

Note the Node.js requirement: webship-js 2.0 needs Node 20 or newer. If `npm install` succeeds but tests crash with syntax errors from inside `node_modules`, an old Node version is the first thing to check.

**At scale:** If this is the third or tenth project you are setting up, stop typing these commands by hand. Put the directory skeleton, a `cucumber.js` template, and the npm scripts into a starter repository (or a small `init` script) and clone it for every new product. The win is not the saved minutes -- it is that every suite in your organization ends up with the same folder layout, so anyone can navigate any project's tests without asking where things live.

### Setup 2: Adding webship-js to an Existing Project

**Problem:** The application already exists -- it might be five years old, written in a framework nobody on the QA team codes in, and shipping every two weeks. You want acceptance tests around it without touching the application's own codebase or build, because the moment your test setup requires changes to the app, you need a developer's time and a release window just to get started.

**Solution:**

```bash
# Navigate to your existing project
cd /path/to/my-existing-project

# Install webship-js
npm install webship-js --save-dev

# Install browsers (if not already installed)
npx playwright install

# Create test directories
mkdir -p tests/features
mkdir -p tests/step-definitions
mkdir -p tests/assets
```

Add test scripts to your existing `package.json`:

```json
{
  "name": "my-existing-project",
  "scripts": {
    "start": "node server.js",
    "test": "cucumber-js",
    "test:smoke": "cucumber-js --tags @smoke",
    "test:full": "cucumber-js --tags 'not @wip'"
  },
  "devDependencies": {
    "webship-js": "^2.0.0"
  }
}
```

**How It Works:**

webship-js lives alongside your application code. The test suite runs against your running application -- it does not import your application code directly. It talks to the app the way a user does: over HTTP, through a real browser. This means webship-js tests work with ANY web application, regardless of the technology it is built with (React, Angular, PHP, Python, Java, etc.).

That black-box boundary is a feature, not a limitation. Because the tests never reach into application internals, they survive refactors, framework migrations, and even full rewrites -- as long as the behavior stays the same, the suite stays green. I have watched teams replace an entire frontend stack underneath a passing acceptance suite. The suite was the proof the rewrite did not break anything users cared about.

**At scale:** Decide early whether tests live inside each application repo or in a separate tests-only repo per project, and apply that decision everywhere. In-repo means tests version together with the code and run on every PR; separate repos suit teams that test products they do not build (an agency QA team, a platform team testing vendor deployments). Both work. Ten projects with five doing it one way and five the other does not.

### Setup 3: Installing Playwright Browsers

**Problem:** Tests pass on your laptop and crash in CI with `browserType.launch: Executable doesn't exist` -- or worse, the browser launches and dies silently because a shared library is missing from the Docker image. It is the classic "works on my machine" of browser automation, and it usually shows up the first time someone wires the suite into a pipeline, on a Friday.

**Solution:**

```bash
# Install all browsers
npx playwright install

# Install only Chromium (fastest, sufficient for most testing)
npx playwright install chromium

# Install Chromium with system dependencies (needed for CI/Docker)
npx playwright install --with-deps chromium

# Install all browsers with system dependencies
npx playwright install --with-deps
```

**How It Works:**

Playwright bundles specific browser versions that are tested against its API. These are not your system browsers -- they are standalone builds that Playwright controls directly.

```
Browser Storage Location:
~/.cache/ms-playwright/
├── chromium-XXXX/
│   └── chrome-linux/chrome
├── firefox-XXXX/
│   └── firefox/firefox
└── webkit-XXXX/
    └── pw_run.sh
```

The `--with-deps` flag installs system-level libraries (fonts, graphics libraries) that browsers need on headless Linux servers. Always use this flag in CI/CD environments.

Because the browser build is pinned to the Playwright version, the browser your test ran against locally is byte-for-byte the browser CI runs -- if your Playwright versions match. That "if" is the catch. A lockfile drift between projects means project A tests on one Chromium build and project B on another, and one of them will eventually hit a rendering difference the other cannot reproduce.

**At scale:** This is a DevOps problem before it is a QA problem. Cache `~/.cache/ms-playwright/` in your CI configuration keyed on the Playwright version -- downloading ~500MB of browsers for 40 nightly suites is real bandwidth and real minutes. Better still, build one base CI image with browsers and dependencies baked in, version it, and point every project's pipeline at it. When you upgrade Playwright, you upgrade the image once and every project moves together instead of drifting one lockfile at a time.

---

## 2.3 Project Structure

A well-organized webship-js project follows this structure:

```
my-app-tests/
│
├── cucumber.js                    # Cucumber configuration
├── package.json                   # Project dependencies
├── package-lock.json              # Dependency lock file
│
├── tests/
│   ├── features/                  # Gherkin feature files
│   │   ├── authentication/
│   │   │   ├── login.feature
│   │   │   ├── registration.feature
│   │   │   └── password-reset.feature
│   │   │
│   │   ├── content/
│   │   │   ├── create-post.feature
│   │   │   ├── edit-post.feature
│   │   │   └── delete-post.feature
│   │   │
│   │   ├── api/
│   │   │   ├── users-api.feature
│   │   │   └── posts-api.feature
│   │   │
│   │   └── smoke/
│   │       └── health-check.feature
│   │
│   ├── step-definitions/          # Custom step definitions
│   │   ├── auth-steps.js          # (only if you need custom steps)
│   │   └── custom-steps.js
│   │
│   └── assets/                    # Test files (images, docs, etc.)
│       ├── test-image.png
│       ├── sample.pdf
│       └── test-data.csv
│
├── reports/                       # Generated test reports
│   └── (auto-generated)
│
└── node_modules/                  # Dependencies (auto-generated)
    └── webship-js/
        └── tests/
            └── step-definitions/
                └── webship.js     # 100+ pre-built steps
```

### Key Principles

1. **Features organized by domain**, not by technical layer. Group by what users care about (authentication, content, shopping), not by what developers care about (controllers, models, views).

2. **Custom steps are optional.** Many projects never need custom step definitions because webship-js covers the common interactions. Only create custom steps when you have domain-specific behaviors.

3. **Assets folder for test files.** Any files your tests need to upload (images, documents, CSVs) go here.

4. **Reports are generated, not committed.** Add `reports/` to your `.gitignore`.

---

## 2.4 Configuration Deep Dive

### Setup 4: The cucumber.js Configuration File

**Problem:** You run `npx cucumber-js` and get `0 scenarios, 0 steps` -- or 47 "undefined step" warnings for steps you know exist. Both mean the same thing: Cucumber does not know where your features and step definitions live. Every webship-js project clears this hurdle exactly once, in one file, and then never thinks about it again -- unless the file is wrong, in which case nothing else in this book works.

**Solution:**

Create `cucumber.js` in your project root:

```javascript
module.exports = {
  default: {
    // Where to find step definitions
    requireModule: [],
    require: [
      'node_modules/webship-js/tests/step-definitions/*.js'
    ],

    // Where to find feature files
    paths: [
      'tests/features/**/*.feature'
    ],

    // webship-js configuration
    worldParameters: {
      launchUrl: 'http://localhost:8080',
      minWaitTime: {
        page: 3000,
        before_scenario: 0,
        after_scenario: 0,
        before_step: 0,
        after_step: 0
      },
      assets_folder: './tests/assets/'
    },

    // Output format
    format: ['progress-bar'],
    publishQuiet: true
  }
};
```

**How It Works:**

The configuration file is a CommonJS module that exports profiles. The `default` profile is used when you run `npx cucumber-js` without specifying a profile.

Let me explain each section:

**requireModule**: Preprocessors to load before step definitions. Empty for standard JavaScript projects. You would add `['ts-node/register']` for TypeScript.

**require**: Paths to step definition files. The glob `*.js` loads all JavaScript files. The order matters -- webship-js steps are loaded first, then your custom steps.

**paths**: Where to find `.feature` files. The `**` glob pattern matches subdirectories at any depth.

**worldParameters**: Configuration passed to the PlaywrightWorld constructor. This is where webship-js gets its settings.

**format**: Output formatters. Options include:
- `progress-bar`: Shows a progress bar (good for CI)
- `progress`: Dots for each step (minimal output)
- `summary`: Just the final summary
- `json:path/to/file.json`: JSON output for reporting tools
- `html:path/to/file.html`: HTML report

**publishQuiet**: Suppresses the "publish your results" prompt from Cucumber.

Two defaults worth knowing because you will hit them eventually. First, the webship-js default profile sets the step timeout to **45000ms** -- any single step that runs longer than 45 seconds fails the scenario, which is your safety net against a hung page stalling the whole suite. Second, the shipped configuration writes a JSON report to `tests/reports/cucumber_report.json` on every run; that file is the raw material the report generator (and your CI dashboard) reads later, so do not be surprised when it appears.

**At scale:** Treat `cucumber.js` as a template, not a per-project artifact. Notice that nothing in the file above is actually project-specific except `launchUrl` -- and that should come from an environment variable anyway (Setup 5). When all 20 of your projects share an identical config file, "how is this suite configured?" has one answer, and a fix (say, adding a formatter) is a copy-paste rollout instead of 20 small archaeology digs.

### Setup 5: Configuring the Base URL

**Problem:** The same suite has to run against a developer's local instance, the staging server, and -- for a careful subset -- production. The first instinct is to edit the config file before each run, and that instinct produces the classic accident: someone commits `launchUrl: 'https://www.example.com'`, the nightly run executes against production, and Monday starts with an incident review. Feature files must never know or care which environment they are pointed at.

**Solution:**

The `launchUrl` in `cucumber.js` sets the default:

```javascript
worldParameters: {
  launchUrl: 'http://localhost:8080'
}
```

Override it per environment with the `LAUNCH_URL` environment variable:

```bash
# Local development
npm test

# Staging environment
LAUNCH_URL=https://staging.example.com npm test

# Production (read-only smoke tests)
LAUNCH_URL=https://www.example.com npm run test:smoke

# Using .env file pattern
export LAUNCH_URL=https://staging.example.com
npm test
```

**How It Works:**

webship-js resolves the base URL in this order:
1. `LAUNCH_URL` environment variable (highest priority)
2. `worldParameters.launchUrl` in `cucumber.js` (default)
3. Built-in fallback: `http://localhost:8080` if neither is set

When you write `Given I am on homepage`, webship-js navigates to this URL. When you write `When I go to "/login"`, it navigates to `{launchUrl}/login`. The resolution happens once, when the PlaywrightWorld is constructed for each scenario -- so the variable must be set when the process starts, not exported mid-run.

This means your feature files are **environment-independent**:

```gherkin
# This works on localhost, staging, AND production
Given I am on homepage
When I go to "/login"
And I fill in "Email" with "test@example.com"
```

Create npm scripts for each environment:

```json
{
  "scripts": {
    "test": "cucumber-js",
    "test:local": "LAUNCH_URL=http://localhost:3000 cucumber-js",
    "test:staging": "LAUNCH_URL=https://staging.example.com cucumber-js",
    "test:prod": "LAUNCH_URL=https://www.example.com cucumber-js --tags @smoke"
  }
}
```

Notice the production script filters to `@smoke` -- read-only checks only. The environment variable controls *where* tests run; tags control *which* tests are safe to run there. You need both halves of that discipline.

**At scale:** `LAUNCH_URL` is the single most important convention in a multi-project setup. In CI, it comes from the pipeline's environment configuration (a GitLab CI variable, a GitHub Actions environment, a Jenkins parameter) -- never from an edited file. That means the exact same suite, the exact same commit, produces the staging result and the production smoke result, and your delivery manager can compare them. The day a release candidate is questioned, "same tests, two URLs, both green" is a sentence everyone in the room understands.

### Setup 6: Environment Variable Overrides

**Problem:** A test fails only in CI. To debug it locally you want Firefox instead of Chromium, a visible browser window, and slowed-down actions you can watch -- but you absolutely do not want to commit any of those settings, because the next nightly run must be back to fast, headless Chromium. Config-file editing for temporary needs is how debug settings leak into main.

**Solution:**

Use environment variables for all dynamic settings:

```bash
# Base URL
export LAUNCH_URL=https://staging.example.com

# Browser choice (chromium / firefox / webkit)
export BROWSER=firefox

# Headed mode ('false' opens a visible browser window)
export HEADLESS=false

# Slow motion in ms (for debugging)
export SLOW_MO=1000

# Run tests
npm test
```

**How It Works:**

These variables are read by `playwright.config.ts` when the browser launches. `BROWSER` accepts `chromium`, `firefox`, or `webkit`. `HEADLESS` is headed only when set to the literal string `'false'`. `SLOW_MO` inserts a pause between every Playwright action; if you do not set it, webship-js picks a sensible default for the context -- 0 in CI (full speed), 300ms when running headless locally, and 800ms in headed mode so your eyes can follow what the browser is doing. Because everything is an environment variable, your debugging session leaves no trace in the repository: close the terminal and the suite is back to its defaults.

**Pro Tip:** Combine them in one line for a debug run: `HEADLESS=false SLOW_MO=1000 BROWSER=firefox npx cucumber-js tests/features/auth/login.feature`. One command, one feature file, watchable speed, gone when you are done.

### Setup 7: Configuring Wait Times

**Problem:** Your application loads in under a second on localhost and in four seconds on the shared staging server every team in the company deploys to. The same suite that is rock solid locally starts failing intermittently on staging -- and the first reflex, sprinkling sleep statements through the scenarios, is exactly the move that turns a fast suite into a slow, still-flaky one.

**Solution:**

```javascript
worldParameters: {
  launchUrl: 'http://localhost:8080',
  minWaitTime: {
    page: 3000,          // Wait after page navigation (ms)
    before_scenario: 0,  // Wait before each scenario
    after_scenario: 0,   // Wait after each scenario
    before_step: 0,      // Wait before each step
    after_step: 0        // Wait after each step
  }
}
```

**Guidelines:**

```
┌─────────────────────┬───────────┬──────────────────────────────────┐
│ Setting             │ Default   │ When to Change                   │
├─────────────────────┼───────────┼──────────────────────────────────┤
│ page                │ 3000ms    │ Increase for slow SPAs,          │
│                     │           │ decrease for static sites        │
├─────────────────────┼───────────┼──────────────────────────────────┤
│ before_scenario     │ 0ms       │ Increase if scenarios need       │
│                     │           │ time to clean up state           │
├─────────────────────┼───────────┼──────────────────────────────────┤
│ after_scenario      │ 0ms       │ Increase if you need to          │
│                     │           │ capture screenshots/logs         │
├─────────────────────┼───────────┼──────────────────────────────────┤
│ before_step         │ 0ms       │ Increase for visual debugging    │
│                     │           │ (watch each step execute)        │
├─────────────────────┼───────────┼──────────────────────────────────┤
│ after_step          │ 0ms       │ Increase for slow applications   │
│                     │           │ that need settling time          │
└─────────────────────┴───────────┴──────────────────────────────────┘
```

**How It Works:**

Here is the part that surprises people coming from older frameworks: these are budgets, not sleeps. The `page: 3000` value is the per-navigation budget for **smartSettle**, webship-js's smart-wait engine. On every page load, an init script installs three counters in the browser: `window.__webshipAjaxCount` (wraps fetch and XHR, counts in-flight requests), `window.__webshipPendingTimers` (wraps setTimeout), and `window.__webshipLastMutation` (a MutationObserver timestamp). After navigation, smartSettle polls those counters every 50ms and declares the page "settled" the moment AJAX hits zero, timers hit zero, and the DOM has been quiet for 250ms -- or gives up when the budget runs out and lets the next step proceed anyway. A static site settles in milliseconds; a chatty SPA uses the full window only when it genuinely needs it. The same engine runs automatically after every state-changing step with a 1500ms budget, which is why most scenarios need no explicit waits at all. If you ever need to prove a timing bug is not smartSettle's doing, disable it with `WEBSHIP_AUTO_SETTLE=off`.

**Pro Tip:** For local development, keep wait times low for fast feedback. For CI against a shared staging server, you might need to increase `page` to 5000-8000ms -- and because it is a budget, the increase costs you nothing on pages that settle quickly. Resist touching `before_step`/`after_step`: those ARE static delays, multiplied by every step in the suite. A 500ms `after_step` on a 2,000-step suite is 16 minutes of pure waiting.

### Setup 8: Setting the Assets Folder

**Problem:** The profile-picture upload test works on your machine because the test image happens to sit in your Downloads folder. The new hire clones the repo, runs the suite, and the upload scenarios fail with file-not-found. Test files are test fixtures -- they belong in the repository, in one agreed place, referenced by relative path.

**Solution:**

```javascript
worldParameters: {
  launchUrl: 'http://localhost:8080',
  assets_folder: './tests/assets/'
}
```

Populate the folder with test files:

```bash
tests/assets/
├── images/
│   ├── test-avatar.png        # 100x100 profile picture
│   ├── large-image.jpg        # Test large file handling
│   └── test-logo.svg          # SVG upload test
├── documents/
│   ├── sample.pdf             # PDF upload test
│   ├── test-spreadsheet.xlsx  # Excel upload test
│   └── readme.txt             # Plain text upload test
└── media/
    ├── test-video.mp4         # Video upload test
    └── test-audio.mp3         # Audio upload test
```

Use them in features:

```gherkin
When I attach the file "images/test-avatar.png" to "profile_picture"
When I attach the file "documents/sample.pdf" to "resume"
```

**Pro Tip:** Keep assets small and synthetic. A 100x100 PNG tests an avatar upload as well as a 4MB photo does, and across hundreds of CI runs the difference shows up in checkout times and runner storage. The exception: keep exactly one deliberately large file for the "rejects oversized uploads" scenario.

### Setup 9: Browser Configuration

**Problem:** A customer files a bug that only happens in Safari, and your suite has been quietly testing Chromium-only since day one. Meanwhile the platform team asks what flags the browser launches with inside their Docker runners, and nobody on the QA side can answer. Before you can debug either conversation, you need to know exactly what webship-js launches by default -- which browser, which mode, which arguments -- and which knobs change it.

**Solution:**

The Playwright configuration can be customized. webship-js uses these defaults:

```
┌──────────────────────────────────────────────────────┐
│ Default Playwright Configuration in webship-js:      │
│                                                      │
│ Browser:     Chromium (BROWSER env to change)        │
│ Headless:    true (HEADLESS='false' for headed)      │
│ Slow Motion: 0 in CI / 300ms headless / 800ms headed │
│ Viewport:    null (uses the real window size)        │
│ HTTPS:       ignoreHTTPSErrors: true                 │
│ Args:        --no-sandbox                            │
│              --disable-setuid-sandbox                │
│              --disable-dev-shm-usage                 │
│              --disable-accelerated-2d-canvas         │
│              --no-first-run                          │
│              --no-zygote                             │
│              --disable-gpu                           │
└──────────────────────────────────────────────────────┘
```

**How It Works:**

Three of these defaults deserve a closer look. The slow-motion value adapts to context: 0 in CI because nobody is watching and speed is money, 300ms for local headless runs, 800ms when headed so a human can follow along -- override any of them with `SLOW_MO`. The `viewport: null` setting means the page gets the browser window's actual size rather than an emulated fixed viewport; when you need specific dimensions, the responsive steps and the named breakpoints (xs through xxxl) handle that explicitly per scenario. And `ignoreHTTPSErrors: true` exists because staging environments live on self-signed certificates -- without it, half your environments would fail at the TLS handshake before the first step ran.

**At scale:** The browser args list is tuned for containerized CI (`--no-sandbox`, `--disable-dev-shm-usage` are the classic fixes for Chromium dying inside Docker with cryptic crashes). If your platform team builds custom runner images, keep these defaults -- removing them is the most common cause of "the suite segfaults only on the new runners."

### Setup 10: Headless vs Headed Mode

**Problem:** A scenario fails and the error message tells you what was missing but not why. You need to watch the browser do the thing -- see the modal that did not close, the spinner that never stopped. But headed mode must stay a local-only habit: CI runners have no display, and a teammate who pulls your branch should never inherit a config that pops browser windows.

**Solution:**

By default, webship-js runs headless (no visible browser). For debugging, you can control this with environment variables or by modifying the Playwright config.

For CI:
```bash
# Headless (default, no visible browser)
npm test
```

For local debugging:
```bash
# Headed mode (opens a visible browser window)
HEADLESS=false npm test
```

Watching the browser execute your steps is incredibly useful when a test fails and you cannot figure out why. You can see exactly what the browser sees, including timing issues and element visibility problems. Headed mode also changes the pace for you: the slow-motion default rises to 800ms between actions, so the run is watchable without you setting anything.

**Pro Tip:** When the failure only happens in CI and you cannot reproduce it headed, do not guess -- turn on video. `WEBSHIP_VIDEO=on-failure` records every scenario and keeps the videos only for the ones that fail. It is the closest thing to standing behind the CI runner watching its screen. Chapter 46 covers video recording in depth.

---

## 2.5 Running Your First Test

### Setup 11: Hello World Feature

**Problem:** You have installed packages, downloaded browsers, and written a config file -- and so far nothing has proven any of it actually works together. Before you invest a day writing real scenarios, you want one two-line test that exercises the whole chain: config, step matching, browser launch, navigation, assertion. If this passes, every layer underneath is wired correctly.

**Solution:**

Create `tests/features/smoke/hello-world.feature`:

```gherkin
Feature: Hello World
  As a developer
  I want to verify webship-js is working
  So that I can start writing real tests

  @smoke
  Scenario: Homepage loads
    Given I am on homepage
    Then I should see "Welcome"
```

Run it:

```bash
npm test
```

**Expected Output (Pass):**

```
1 scenario (1 passed)
2 steps (2 passed)
0m4.523s
```

**Expected Output (Fail):**

```
1 scenario (1 failed)
2 steps (1 passed, 1 failed)

Failing steps:
  Then I should see "Welcome"
  
  AssertionError: Expected to find "Welcome" on the page
    at World.Then (/node_modules/webship-js/tests/step-definitions/webship.js:XXX)
```

If the test fails, check:
1. Is your application running at the `launchUrl`?
2. Does the homepage actually contain the text "Welcome"?
3. Is the URL correct?

**How It Works:**

Those two innocent lines exercise more machinery than they let on. `Given I am on homepage` makes Cucumber construct a fresh PlaywrightWorld for the scenario, launch the browser, open a new context and page, navigate to the resolved `launchUrl`, and run smartSettle until the page goes quiet (or the 3000ms navigation budget elapses). `Then I should see "Welcome"` then asserts against the visible page text. Roughly four seconds for two steps sounds slow until you realize most of it is one-time browser startup -- additional scenarios in the same run reuse the launched browser and are much cheaper.

### Setup 12: Understanding Test Output

**Problem:** The first time a teammate runs the suite, they read `12 steps (10 passed, 1 failed, 1 skipped)` and ask you whether "skipped" is something they broke. Output literacy sounds trivial until you watch someone spend twenty minutes investigating a skipped step that was never the problem -- the failure one line above it was.

**Solution:**

Cucumber provides several output formats. Here is what each tells you:

**Progress bar format** (default):

```
Feature: Login ..F..

Failures:
1) Scenario: Failed login
   Step: Then I should see "Invalid credentials"
   Error: Expected "Invalid credentials" but found "Error occurred"

3 scenarios (2 passed, 1 failed)
12 steps (10 passed, 1 failed, 1 skipped)
0m12.456s
```

Understanding the symbols:
```
.  = Passed step
F  = Failed step
-  = Skipped step (after a failure)
P  = Pending step (step definition exists but not implemented)
U  = Undefined step (no step definition found)
```

### Setup 13: Reading Error Messages

**Problem:** It is 4:50 p.m., the pipeline is red, and the error says `Expected to find 'Dashboard' on the page`. Is the application broken? Is the test wrong? Is staging down? Every minute spent decoding the message is a minute the release waits. The good news: webship-js failures cluster into a handful of patterns, and each one points at a different culprit.

**Solution:**

Common error messages and their meanings:

```
┌────────────────────────────────────────────────────────────────┐
│ Error: "Expected to find 'X' on the page"                     │
│ Cause: The text X does not appear on the visible page.        │
│ Fix:   - Use a web-first assertion ("should have text")       │
│          with a "within N seconds" budget for slow content    │
│        - Check the spelling of the expected text               │
│        - Check that the content is visible (not hidden)        │
├────────────────────────────────────────────────────────────────┤
│ Error: "Element not found: [selector]"                         │
│ Cause: Playwright cannot find the element on the page.        │
│ Fix:   - Check the field label/name matches                   │
│        - Wait for the element ("should be visible") first      │
│        - Verify the page loaded correctly                      │
├────────────────────────────────────────────────────────────────┤
│ Error: "Navigation timeout of 30000ms exceeded"                │
│ Cause: The page did not load within 30 seconds.               │
│ Fix:   - Check the URL is correct                              │
│        - Check the application is running                      │
│        - Increase timeout if the page is legitimately slow     │
├────────────────────────────────────────────────────────────────┤
│ Error: "Step 'X' is undefined"                                 │
│ Cause: No step definition matches the step text.              │
│ Fix:   - Check spelling matches a webship-js step exactly     │
│        - Create a custom step definition                       │
│        - Check the require path in cucumber.js                 │
├────────────────────────────────────────────────────────────────┤
│ Error: "net::ERR_CONNECTION_REFUSED"                           │
│ Cause: Cannot connect to the launchUrl.                       │
│ Fix:   - Start your application server                         │
│        - Check the port number matches                         │
│        - Check no firewall is blocking the connection          │
└────────────────────────────────────────────────────────────────┘
```

**How It Works:**

The diagnostic order matters. Connection refused and navigation timeouts are environment problems -- the application or network, not your test. Undefined steps are wiring problems -- your `cucumber.js` require paths or a typo in the step text. Only the first two ("expected to find", "element not found") are genuine test-versus-application questions, and webship-js answers most of them for you: every failed step automatically captures a screenshot (`WEBSHIP_SCREENSHOT_ON_FAILED` defaults to `true`, saved with a `failed_` prefix under `./screenshots`). Open the screenshot before you open the code -- nine times out of ten it shows an error page, a still-loading spinner, or a cookie-consent overlay sitting exactly where your element should be.

**At scale:** Teach this triage order to every project team, because it determines who gets paged. Connection errors go to DevOps, undefined steps go to whoever changed the test code, assertion failures go to QA plus the feature developer. A failure-routing convention sounds bureaucratic until the morning 6 of your 40 nightly suites are red and all six turn out to be one staging outage -- which a thirty-second look at the error class would have told you.

---

## 2.6 Multiple Configuration Profiles

### Setup 14: Named Profiles for Different Scenarios

**Problem:** The same suite serves three masters with three clocks. The developer pushing a hotfix wants a two-minute smoke signal. The nightly job wants everything except work-in-progress, with reports. The API checks should run on every commit because they need no browser and finish in seconds. One configuration cannot satisfy all three, and maintaining three config files means the day someone fixes a path in one, the other two quietly rot.

**Solution:**

```javascript
// cucumber.js
const common = {
  requireModule: [],
  require: [
    'node_modules/webship-js/tests/step-definitions/*.js',
    'tests/step-definitions/*.js'
  ],
  worldParameters: {
    launchUrl: process.env.LAUNCH_URL || 'http://localhost:8080',
    assets_folder: './tests/assets/'
  },
  publishQuiet: true
};

module.exports = {
  // Default: run everything
  default: {
    ...common,
    paths: ['tests/features/**/*.feature'],
    format: ['progress-bar']
  },

  // Smoke: quick health checks
  smoke: {
    ...common,
    paths: ['tests/features/**/*.feature'],
    tags: '@smoke',
    format: ['progress-bar']
  },

  // API: only API tests (no browser needed)
  api: {
    ...common,
    paths: ['tests/features/api/**/*.feature'],
    format: ['progress-bar']
  },

  // CI: full suite with reports
  ci: {
    ...common,
    paths: ['tests/features/**/*.feature'],
    tags: 'not @wip',
    format: [
      'progress-bar',
      'json:reports/cucumber-report.json',
      'html:reports/cucumber-report.html'
    ]
  }
};
```

Run specific profiles:

```bash
# Default profile
npx cucumber-js

# Smoke tests
npx cucumber-js --profile smoke

# API tests only
npx cucumber-js --profile api

# CI profile with reports
npx cucumber-js --profile ci
```

Add to `package.json`:

```json
{
  "scripts": {
    "test": "cucumber-js",
    "test:smoke": "cucumber-js --profile smoke",
    "test:api": "cucumber-js --profile api",
    "test:ci": "cucumber-js --profile ci"
  }
}
```

**How It Works:**

The `common` object plus spread syntax (`...common`) is the whole trick: shared settings live in one place, and each profile only declares what makes it different -- a tag filter, a path restriction, extra formatters. Note how the `ci` profile adds the JSON formatter: that JSON file is what report generators consume, so CI is the profile where it belongs. Filtering by tags rather than file paths (the `smoke` profile scans all features but runs only `@smoke` scenarios) means a scenario's membership in the smoke suite is declared next to the scenario itself, where reviewers can see it.

**At scale:** Standardize the profile names across every project before you standardize anything else. When `npm run test:smoke` means the same thing in all 20 repos, your CI templates become trivial -- one pipeline definition, parameterized by project, instead of 20 bespoke YAML files. It also makes the cross-project dashboard honest: "smoke" is comparable across products only if every product defines it the same way. The day a new project joins, it inherits the profiles, the scripts, and the pipeline in one copy.

---

## Chapter Summary

- webship-js = Cucumber-js + Playwright + 411 pre-built step definitions in 36 step files
- Installation is three commands: `npm install`, `npx playwright install`, create config
- The `cucumber.js` config file controls paths, URLs, wait times, and output
- `launchUrl` is the most important setting -- override with `LAUNCH_URL` env var
- Feature files go in `tests/features/`, organized by domain area
- Each scenario gets a fresh browser context (complete isolation)
- Use named profiles for different testing scenarios (smoke, full, api, ci)

In the next chapter, we will master Gherkin -- the language that makes all of this possible.

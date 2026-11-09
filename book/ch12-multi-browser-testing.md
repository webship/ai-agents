# Chapter 12: Multi-Browser Testing

---

Your users do not all use the same browser. Some are on Chrome, others on Firefox, some on Safari, and a surprising number on Edge (which runs on Chromium). A feature that works perfectly in Chrome might break in Firefox due to a CSS rendering difference or in Safari due to a missing JavaScript API.

Multi-browser testing catches these compatibility bugs before your users do. And with webship-js and Playwright, running your entire test suite across three browser engines is as simple as changing an environment variable.

Here is the uncomfortable part: most teams discover their Safari bugs from support tickets, because everyone develops in Chrome and the test suite runs in Chrome too. The fix is not heroic effort -- it is making the other two engines a routine part of the schedule, which this chapter shows you how to do without touching a single feature file.

---

## Understanding Browser Engines

```
┌──────────────────────────────────────────────────────────────────┐
│ Browser Engines and the Browsers They Power                      │
│                                                                  │
│  ┌──────────────────────┐                                       │
│  │ Chromium (Blink)     │                                       │
│  │                      │   Browsers:                            │
│  │ Google's open-source │   • Google Chrome                     │
│  │ browser engine       │   • Microsoft Edge                    │
│  │                      │   • Opera                             │
│  │ ~65% market share    │   • Brave                             │
│  │                      │   • Vivaldi                            │
│  └──────────────────────┘   • Samsung Internet                  │
│                                                                  │
│  ┌──────────────────────┐                                       │
│  │ Firefox (Gecko)      │                                       │
│  │                      │   Browsers:                            │
│  │ Mozilla's engine     │   • Mozilla Firefox                   │
│  │                      │   • Tor Browser                       │
│  │ ~3% market share     │   • LibreWolf                         │
│  └──────────────────────┘                                       │
│                                                                  │
│  ┌──────────────────────┐                                       │
│  │ WebKit               │                                       │
│  │                      │   Browsers:                            │
│  │ Apple's engine       │   • Safari (macOS, iOS)               │
│  │                      │   • All iOS browsers (due to Apple    │
│  │ ~19% market share    │     policy, all iOS browsers use      │
│  │ (mostly mobile)      │     WebKit under the hood)            │
│  └──────────────────────┘                                       │
│                                                                  │
│  Playwright tests against these three engines, which covers     │
│  virtually 100% of real-world browsers.                          │
└──────────────────────────────────────────────────────────────────┘
```

By testing against Chromium, Firefox, and WebKit, you cover every major browser engine. This is one of Playwright's key advantages over tools that only support Chromium (like Puppeteer). One engine per family is enough: Chrome, Edge, Opera, and Brave all render with Blink, so a Chromium pass covers them all. And the market-share numbers above are global averages -- the numbers that matter are in your own analytics. A public-facing site can easily see 30% Safari traffic while an internal admin tool sees almost none. Let that data decide how much weight each engine gets in your schedule.

---

## How webship-js Handles Browser Selection

webship-js uses a `BROWSER` environment variable to select which browser engine to use. The configuration is in `playwright.config.ts`:

```typescript
type BrowserName = 'chromium' | 'firefox' | 'webkit';

const browser = (process.env.BROWSER || 'chromium') as BrowserName;
const headless = process.env.HEADLESS !== 'false';

// SLOW_MO default adapts to where the suite runs:
// 0 in CI, 300ms headless on a laptop, 800ms when headed.
const defaultSlowMo = process.env.CI ? 0 : headless ? 300 : 800;

const chromiumArgs: string[] = [
  '--no-sandbox',
  '--disable-dev-shm-usage',
  '--disable-web-security',
  '--ignore-certificate-errors',
  '--incognito',
  '--start-maximized',
];

const config = {
  browser,
  launchOptions: {
    headless,
    slowMo: parseInt(process.env.SLOW_MO || String(defaultSlowMo), 10),
    args: browser === 'chromium' ? chromiumArgs : [],
  },
  contextOptions: {
    viewport: null,
    ignoreHTTPSErrors: true,
  },
};
```

Key points:
- Default browser is `chromium` if `BROWSER` is not set
- Chromium gets special launch arguments (no sandbox, ignore certificate errors, etc.)
- Firefox and WebKit use default arguments
- All browsers respect `HEADLESS` and `SLOW_MO`; the `SLOW_MO` default adapts to context -- 0 in CI, 300ms headless, 800ms headed
- `viewport: null` means the page uses the real window size instead of a fixed emulated one
- `ignoreHTTPSErrors: true` works across all browsers

The variable is read once at launch -- one process, one engine. That single line of indirection is what keeps multi-browser testing cheap: your features, step definitions, and selectors never branch on the browser, so the cost of adding an engine is CI minutes, not maintenance.

---

## Recipe 134: Testing on Chromium (Default)

**Problem:** It is your first week wiring tests into a new project, and you need the fastest, most predictable engine to build on. Chromium is where you will write scenarios, debug failures, and gate pull requests -- the other engines come later. Before you start trusting green checkmarks, you want to know exactly what the default run does and why it behaves differently inside a container than on your laptop.

**Solution:**

```bash
# Default -- no environment variable needed
npx cucumber-js

# Explicit Chromium
BROWSER=chromium npx cucumber-js

# Run specific feature on Chromium
BROWSER=chromium npx cucumber-js tests/features/login.feature
```

```gherkin
# This feature runs identically on any browser
Feature: Login on Chromium
  Scenario: Successful login
    Given I am on homepage
    When I go to "/login"
    And I fill in "Email" with "admin@example.com"
    And I fill in "Password" with "AdminPass123"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "Dashboard"
```

**Chromium-specific behavior:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Chromium Launch Arguments                                        │
│                                                                  │
│ --no-sandbox           Runs without sandbox (needed in Docker/CI)│
│ --disable-dev-shm-usage  Prevents crashes on low-memory systems │
│ --disable-web-security   Allows cross-origin requests in tests  │
│ --ignore-certificate-errors  Ignores SSL cert issues (dev certs)│
│ --incognito            Clean browser state for each context     │
│ --start-maximized      Full-size viewport                       │
│                                                                  │
│ These arguments are only applied to Chromium, not Firefox/WebKit │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** When the suite starts, `playwright.config.ts` reads the `BROWSER` environment variable once and falls back to `chromium` when it is unset -- so `npx cucumber-js` and `BROWSER=chromium npx cucumber-js` are the exact same run. Chromium is the only engine that gets the extra launch arguments above, and they exist for one reason: Chromium is the engine you will run inside Docker and CI containers, where sandboxing restrictions and tiny `/dev/shm` partitions bite hardest. Everything else about the run is engine-independent. The same built-in steps execute, and smartSettle still watches `window.__webshipAjaxCount`, pending timers, and DOM mutations before letting each step proceed. Your feature files have no idea which browser they are driving -- that is the whole point.

**At scale:** Make Chromium the pull-request gate on every project, and keep the command identical everywhere: `npx cucumber-js --tags "@smoke"`, no per-project flags. When all twenty suites share the same default, a QA lead can jump between projects without reading each repo's CI file first, and the platform team reasons about one container setup instead of twenty. Save Firefox and WebKit for the nightly schedule -- Recipe 137 covers how.

---

## Recipe 135: Testing on Firefox

**Problem:** A user reports that your date picker shows "Invalid Date" -- but only in Firefox. Nobody on the team runs Firefox, the suite only runs Chromium, and it turns out the bug has been live for three weeks. You need Firefox in the rotation, so the next Gecko-only regression shows up in a test report instead of a support ticket.

**Solution:**

```bash
# Run all tests on Firefox
BROWSER=firefox npx cucumber-js

# Run specific feature on Firefox
BROWSER=firefox npx cucumber-js tests/features/login.feature

# Run tagged tests on Firefox
BROWSER=firefox npx cucumber-js --tags "@smoke"
```

**Firefox-specific considerations:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Firefox Differences to Watch For                                 │
│                                                                  │
│ 1. CSS Rendering:                                                │
│    - Scrollbar styling differs (Firefox uses overlay scrollbars  │
│      on some platforms)                                          │
│    - Flexbox gap rendering can differ                            │
│    - backdrop-filter may not be fully supported                  │
│                                                                  │
│ 2. JavaScript:                                                   │
│    - Some Web APIs may be behind flags                           │
│    - Date parsing differences (e.g., "2024-01-15" vs "Jan 15")  │
│                                                                  │
│ 3. Form Elements:                                                │
│    - File input styling is different                              │
│    - Autofill behavior differs                                   │
│    - Date/time input widgets look different                      │
│                                                                  │
│ 4. Network:                                                      │
│    - CORS handling can have subtle differences                   │
│    - Cookie SameSite defaults may differ                         │
│                                                                  │
│ webship-js handles most of these transparently through           │
│ Playwright's unified API.                                        │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** Setting `BROWSER=firefox` makes Playwright launch its own bundled Firefox build -- not whatever copy happens to be installed on the machine -- so every developer and every CI runner drives the exact same Gecko version. The Chromium-only launch arguments are skipped automatically; Firefox neither understands nor needs them. Every step behaves identically: `I fill in "Email"` still resolves the field label-first, web-first assertions still poll every 100ms for up to 5 seconds, and smartSettle still waits for AJAX and DOM quiet after every state-changing step. So when a scenario passes on Chromium and fails on Firefox, the difference is almost never the test -- it is genuinely your application rendering or parsing something differently, which is exactly the information you wanted.

**Pro Tip:** When a Firefox failure looks mysterious, rerun just that feature headed: `BROWSER=firefox HEADLESS=false npx cucumber-js tests/features/login.feature`. Watching Gecko render the page usually exposes the difference -- a wrapped flexbox item, a differently parsed date -- in seconds.

---

## Recipe 136: Testing on WebKit (Safari)

**Problem:** Your analytics say 19% of visitors are on Safari, and a good chunk of them are on iPhones -- where every browser, including "Chrome for iOS", is WebKit underneath. You have no Mac in the CI fleet and nobody clicking through pages on an iPhone before each release. WebKit coverage has quietly become your biggest blind spot.

**Solution:**

```bash
# Run all tests on WebKit
BROWSER=webkit npx cucumber-js

# Run specific feature on WebKit
BROWSER=webkit npx cucumber-js tests/features/login.feature
```

**WebKit-specific considerations:**

```
┌──────────────────────────────────────────────────────────────────┐
│ WebKit (Safari) Differences to Watch For                         │
│                                                                  │
│ 1. CSS:                                                          │
│    - -webkit- prefixes still needed for some properties          │
│    - position: sticky behavior can differ                        │
│    - Smooth scrolling may behave differently                     │
│                                                                  │
│ 2. JavaScript:                                                   │
│    - Some newer APIs may not be available                        │
│    - Regex lookbehind support varies                              │
│    - ResizeObserver has had historical bugs                       │
│                                                                  │
│ 3. Network:                                                      │
│    - Stricter third-party cookie blocking (ITP)                  │
│    - More aggressive cache behavior                               │
│    - Service Worker support differences                          │
│                                                                  │
│ 4. Media:                                                        │
│    - Different codec support for video/audio                     │
│    - Autoplay policies are stricter                               │
│                                                                  │
│ Testing on WebKit catches iOS Safari bugs, which is critical     │
│ because ALL iOS browsers use WebKit under the hood.              │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** Playwright ships a WebKit build that runs on Linux and Windows, not just macOS -- that is the trick that makes Safari coverage possible in an ordinary CI container with no Apple hardware in sight. It is the same engine that powers Safari, so the rendering quirks, ITP cookie restrictions, and stricter autoplay rules listed above show up in your test run the way they would on a real device. One practical caveat: on Linux, WebKit needs more system libraries than the other engines, so install it with `npx playwright install --with-deps webkit` (Recipe 142) or run inside the official Playwright Docker image where everything is preinstalled.

**At scale:** WebKit earns its CI minutes unevenly. A consumer-facing site with heavy mobile traffic should run it nightly on the full suite; an internal back-office tool used on managed Chrome desktops might only need a weekly WebKit smoke run. Decide per project from traffic data, write the decision into the project's CI file, and revisit it quarterly. A blanket "all browsers on everything" policy is how nightly pipelines balloon into three hours of runtime nobody looks at.

---

## Recipe 137: Running Tests Across All Browsers

**Problem:** The release retro has the same line item two sprints in a row: a bug users found in Firefox, in a flow your Chromium suite covers and passes. The tests were never the problem -- the schedule was, because nothing ever ran them on the other two engines. You need one command that sweeps Chromium, Firefox, and WebKit and gives you a per-engine verdict, without maintaining three copies of anything.

**Solution:**

```bash
# Run on all browsers sequentially
BROWSER=chromium npx cucumber-js && \
BROWSER=firefox npx cucumber-js && \
BROWSER=webkit npx cucumber-js
```

**Create a shell script for convenience:**

```bash
#!/bin/bash
# run-all-browsers.sh

echo "=== Running tests on Chromium ==="
BROWSER=chromium npx cucumber-js "$@"
CHROMIUM_EXIT=$?

echo "=== Running tests on Firefox ==="
BROWSER=firefox npx cucumber-js "$@"
FIREFOX_EXIT=$?

echo "=== Running tests on WebKit ==="
BROWSER=webkit npx cucumber-js "$@"
WEBKIT_EXIT=$?

echo ""
echo "=== Results ==="
echo "Chromium: $([ $CHROMIUM_EXIT -eq 0 ] && echo 'PASS' || echo 'FAIL')"
echo "Firefox:  $([ $FIREFOX_EXIT -eq 0 ] && echo 'PASS' || echo 'FAIL')"
echo "WebKit:   $([ $WEBKIT_EXIT -eq 0 ] && echo 'PASS' || echo 'FAIL')"

# Exit with failure if any browser failed
exit $(( CHROMIUM_EXIT + FIREFOX_EXIT + WEBKIT_EXIT ))
```

**npm scripts approach:**

```json
{
  "scripts": {
    "test": "cucumber-js",
    "test:chromium": "BROWSER=chromium cucumber-js",
    "test:firefox": "BROWSER=firefox cucumber-js",
    "test:webkit": "BROWSER=webkit cucumber-js",
    "test:all-browsers": "npm run test:chromium && npm run test:firefox && npm run test:webkit"
  }
}
```

```
Multi-Browser Test Results:
┌──────────┬────────┬────────┬────────┬──────────────────────────┐
│ Feature  │ Chrome │ Firefox│ WebKit │ Notes                    │
├──────────┼────────┼────────┼────────┼──────────────────────────┤
│ Login    │   ✓    │   ✓    │   ✓    │ Works everywhere         │
│ Forms    │   ✓    │   ✓    │   ✓    │ Works everywhere         │
│ Modals   │   ✓    │   ✓    │   ✗    │ WebKit animation issue   │
│ Tables   │   ✓    │   ✓    │   ✓    │ Works everywhere         │
│ API      │   ✓    │   ✓    │   ✓    │ No browser dependency    │
│ Upload   │   ✓    │   ✗    │   ✓    │ Firefox file path issue  │
└──────────┴────────┴────────┴────────┴──────────────────────────┘
```

**How It Works:** `BROWSER` is read once at launch, so covering three engines means three separate runs -- the script simply chains them. Two details make it more than a throwaway loop. First, `"$@"` forwards whatever arguments you pass straight into every run, so `./run-all-browsers.sh --tags "@smoke"` filters all three engines the same way. Second, each exit code is captured separately and summed at the end: the script never stops at the first red engine, so a WebKit failure still leaves you with complete Chromium and Firefox results. That completeness is the point -- the matrix above is the real artifact of a multi-browser run, because the failure pattern tells you what kind of bug you have. One engine red means a compatibility bug; all three red means the feature itself is broken. One caveat: every run writes the same JSON report path, so generate or archive the report between runs -- `bin/generate-reports.js --browser firefox` stamps the engine into the report metadata so the three results stay distinguishable afterward.

**At scale:** The sequential script is for laptops. In CI, run the three engines as parallel matrix jobs -- wall time stays flat and you pay the same total minutes either way. Across twenty projects, keep the engine list in the shared pipeline template, not in each repo: when WebKit moves from weekly to nightly, that is one template change instead of twenty pull requests. And always pass `--browser` into the report generator -- a cross-project dashboard that cannot tell a Firefox run from a Chromium run is just noise.

---

## Recipe 138: Headless vs Headed Mode

**Problem:** A scenario fails on the third click and the step output tells you nothing you can use -- by the time you read it, the browser is gone. You want to watch the run: see the page, the click, the menu that never opened. But the same suite has to run on CI machines that have no display at all, so visibility must be a runtime switch, not a code change.

**Solution:**

```bash
# Headless (default) -- no browser window visible
npx cucumber-js

# Headed -- browser window opens, you can watch the test
HEADLESS=false npx cucumber-js

# Headed with slow motion for debugging
HEADLESS=false SLOW_MO=1000 npx cucumber-js

# In CI (always headless)
HEADLESS=true BROWSER=chromium npx cucumber-js
```

**How It Works:**

```
┌──────────────────────────────────┬──────────────────────────────┐
│ Headless Mode (HEADLESS=true)    │ Headed Mode (HEADLESS=false) │
│ (Default)                        │                              │
│                                  │                              │
│ • No browser window visible     │ • Browser window opens       │
│ • Faster execution              │ • Slower (rendering visible) │
│ • Uses less memory              │ • Great for debugging        │
│ • Perfect for CI/CD             │ • Watch tests run in real     │
│ • Default in webship-js         │   time                       │
│                                  │ • See exactly what the test  │
│                                  │   sees                       │
│                                  │                              │
│ Use for:                         │ Use for:                     │
│ • CI/CD pipelines              │ • Writing new tests          │
│ • Automated test runs           │ • Debugging failures         │
│ • Quick test execution          │ • Demos to stakeholders      │
└──────────────────────────────────┴──────────────────────────────┘
```

The switch is the `HEADLESS` environment variable, read by `playwright.config.ts` at launch: the literal string `'false'` opens a real browser window, anything else -- including unset -- stays headless. Headed mode also changes pace. When you have not set `SLOW_MO` yourself, webship-js picks 800ms between actions for headed runs instead of the headless 300ms, because a browser window you cannot follow is no more useful than no window at all. Everything else -- steps, waits, assertions, screenshots -- behaves identically in both modes; headless runs the same engine, just without painting to a screen.

---

## Recipe 139: Controlling Browser Speed

**Problem:** You are demoing the suite to your product owner and the browser flashes through the checkout flow faster than anyone can narrate it. The next morning, the same suite needs every one of those milliseconds back, because CI runs it forty times before lunch. One knob has to serve both audiences -- without anyone editing a config file in between.

**Solution:**

```bash
# Default adapts to context: 0 in CI, 300ms headless, 800ms headed
npx cucumber-js

# No delay (fastest, for CI)
SLOW_MO=0 npx cucumber-js

# Slow for demonstration
SLOW_MO=1000 npx cucumber-js

# Very slow for debugging
HEADLESS=false SLOW_MO=2000 npx cucumber-js
```

**How It Works:**

`SLOW_MO` sets the Playwright `slowMo` option, which adds a delay (in milliseconds) before every Playwright action (click, type, navigate). Combined with `HEADLESS=false`, it lets you watch the test run at a pace you can follow.

You rarely need to set it, because the default already adapts to where the suite runs: 0 when `CI` is set (machines do not watch), 300ms for local headless runs, 800ms for headed ones -- and an explicit `SLOW_MO` always overrides all three. One thing slow motion is not: a fix for flaky tests. It slows every action equally, which can mask a race condition on your laptop and leave it to resurface in CI at `SLOW_MO=0`. Waiting is smartSettle's job -- it holds each step until in-flight requests drain and the DOM goes quiet -- so if a scenario only passes with slow motion, treat that as a missing assertion, not a missing delay.

```
Speed Settings:
┌────────────────────────────────────────────────────────────────┐
│ SLOW_MO Value │ Speed     │ Use Case                          │
├────────────────────────────────────────────────────────────────┤
│ 0             │ Fastest   │ CI/CD pipelines (CI default)      │
│ 100           │ Fast      │ Quick local runs                  │
│ 300           │ Normal    │ Default for local headless        │
│ 500           │ Moderate  │ Watching tests run                │
│ 800           │ Deliberate│ Default for headed runs           │
│ 1000          │ Slow      │ Demos and presentations           │
│ 2000          │ Very slow │ Step-by-step debugging            │
└────────────────────────────────────────────────────────────────┘
```

---

## Recipe 140: Mobile Browser Emulation

**Problem:** Analytics says more than half your visitors arrive on phones, yet every test you own runs in a desktop-sized window. The bugs that matter on mobile -- a hamburger menu that will not open, a checkout button pushed below the fold by a banner -- are invisible at 1920px wide. You do not have a drawer full of test devices, and you do not need one to catch this class of failure.

**Solution:**

Playwright supports device emulation by setting viewport size and user agent. You can configure this in your project:

```javascript
// playwright.config.ts (custom override)
const devices = require('playwright').devices;

// Example: iPhone 13 emulation
const config = {
  browser: 'webkit',  // Safari engine for iOS
  launchOptions: {
    headless: true,
  },
  contextOptions: {
    ...devices['iPhone 13'],
    ignoreHTTPSErrors: true,
  },
};
```

**Testing mobile-specific behavior in Gherkin:**

```gherkin
Feature: Mobile Experience
  Scenario: Hamburger menu on mobile
    Given I am on homepage
    # On mobile viewport, navigation is collapsed
    Then I should not see "Products"
    When I press "Menu"
    Then I should see "Products"
    And I should see "About"
    And I should see "Contact"
    When I click "Products"
    And wait for AJAX to finish
    Then I should see "Our Products"

  Scenario: Mobile-optimized layout
    Given I am on homepage
    Then I should see "Welcome"
    # Mobile layout shows cards stacked vertically
    And I should see "Featured Products"
```

**Device Emulation Profiles:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Common Device Profiles                                           │
│                                                                  │
│ Device          │ Width │ Height │ Scale │ Engine  │ Touch      │
│ ────────────────┼───────┼────────┼───────┼─────────┼───────     │
│ iPhone SE       │  375  │  667   │  2x   │ WebKit  │ Yes        │
│ iPhone 13       │  390  │  844   │  3x   │ WebKit  │ Yes        │
│ iPhone 15 Pro   │  393  │  852   │  3x   │ WebKit  │ Yes        │
│ iPad Pro 11     │  834  │ 1194   │  2x   │ WebKit  │ Yes        │
│ Pixel 7         │  412  │  915   │  2.6x │ Chromium│ Yes        │
│ Galaxy S23      │  360  │  780   │  3x   │ Chromium│ Yes        │
│ Desktop 1080p   │ 1920  │ 1080   │  1x   │ Any     │ No         │
│ Desktop 1440p   │ 2560  │ 1440   │  1x   │ Any     │ No         │
│ Laptop          │ 1366  │  768   │  1x   │ Any     │ No         │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** Spreading `devices['iPhone 13']` into `contextOptions` swaps webship-js's default `viewport: null` for the full device description Playwright maintains: viewport size, `deviceScaleFactor`, `userAgent`, `isMobile`, and `hasTouch`. Your application genuinely sees a phone -- the meta-viewport math, the UA-sniffing code paths, and the touch handlers all engage. Pair the profile with the matching engine: iPhone and iPad profiles belong on `BROWSER=webkit`, because every browser on iOS runs WebKit underneath, while Pixel and Galaxy profiles belong on Chromium. Notice the menu scenario carries no sleep after `I press "Menu"` -- webship-js runs its settle check after every state-changing step, waiting for the open animation's DOM mutations to go quiet for 250ms before the next assertion fires, so the menu is actually open when `I should see "Products"` runs. Be honest about what emulation gives you, though: correct geometry, user agent, and touch flags -- not a real device's CPU, network, or rendering quirks. That covers layout and behavior bugs, which is most of them. For systematic width-by-width testing against your CSS breakpoints, Chapter 13 picks this up properly.

---

## Recipe 141: Browser-Specific Tags

**Problem:** WebKit has a known rendering bug in your modal animation -- the fix is upstream, months away -- and meanwhile every nightly run shows red on a failure nobody is going to act on. A permanently red lane trains the team to stop reading the report, which is worse than having no report. You need a way to park known engine-specific failures without deleting the coverage or losing track of the debt.

**Solution:**

```gherkin
Feature: Cross-Browser Compatibility
  @chromium
  Scenario: Chrome-specific feature
    Given I am on homepage
    # Test something Chrome-specific

  @firefox
  Scenario: Firefox-specific behavior
    Given I am on homepage
    # Test something Firefox-specific

  @skip-webkit
  Scenario: Known WebKit issue
    Given I am on homepage
    # This test fails on WebKit due to a known rendering bug
```

**Run with browser-specific tags:**

```bash
# Run only Chromium-specific tests
BROWSER=chromium npx cucumber-js --tags "@chromium"

# Run all tests EXCEPT WebKit-skipped ones
BROWSER=webkit npx cucumber-js --tags "not @skip-webkit"

# Run cross-browser suite (exclude browser-specific tests)
BROWSER=firefox npx cucumber-js --tags "not @chromium and not @webkit"
```

**How It Works:** There is no magic in these tags -- webship-js does not read `@skip-webkit` and skip anything on its own. They are ordinary Cucumber tags, and the pairing happens on the command line: you pick the engine with `BROWSER` and the tag expression with `--tags`, and the convention only holds if your CI files apply both together, every time. That manual link is actually a strength. Tag expressions compose (`not @skip-webkit and @smoke`), and the tag sits in the feature file where a reviewer will see it and ask why. Treat every skip tag as a ledger entry, not a verdict: each one should carry a comment with the issue link, so that `grep -r "@skip-webkit" tests/` returns an open bug list instead of archaeology.

**At scale:** Agree on one tag vocabulary across all your projects -- `@skip-webkit` must mean exactly the same thing in every repo, or the nightly matrix becomes unreadable. The count of skip tags per project is one of the most honest health metrics a QA lead can put on a dashboard: it is the browser debt you have decided to live with, and it should trend toward zero. Review it monthly. A skip tag that survives three reviews without an issue number attached is a test you have silently deleted.

---

## Recipe 142: Installing Browser Binaries

**Problem:** The suite that ran fine yesterday greets the new CI runner with `browserType.launch: Executable doesn't exist` -- and the new hire hits the same wall on their first `npx cucumber-js`, fifteen minutes into day one. Playwright does not drive whatever Chrome happens to be installed on the machine; it drives its own pinned builds, and those have to be put there. Where they come from, where they live, and which system libraries they drag along is worth understanding once, because every machine you ever onboard repeats the same dance.

**Solution:**

```bash
# Install all browsers
npx playwright install

# Install specific browser
npx playwright install chromium
npx playwright install firefox
npx playwright install webkit

# Install with system dependencies (Linux CI)
npx playwright install --with-deps chromium
npx playwright install --with-deps firefox
npx playwright install --with-deps webkit
```

**CI/CD Installation:**

```yaml
# GitHub Actions example
- name: Install Playwright Browsers
  run: npx playwright install --with-deps

# Docker example
FROM mcr.microsoft.com/playwright:v1.58.0-jammy
WORKDIR /app
COPY . .
RUN npm ci
```

**How It Works:** `npx playwright install` downloads engine builds pinned to the exact Playwright version in your `package.json` -- upgrade Playwright and the next install fetches matching browsers, which is how every laptop and runner ends up driving the same build instead of six slightly different ones. On Linux the binaries land in `~/.cache/ms-playwright`, which is what your CI cache step should key on (put the Playwright version in the cache key, or a framework upgrade will restore stale browsers). The `--with-deps` flag additionally installs the system libraries the engines need -- a near no-op for Chromium on most images, essential for WebKit, which leans on a long list of GTK and media packages a bare container will not have. The Docker route skips the whole question: the `mcr.microsoft.com/playwright:v1.58.0-jammy` image ships all three engines and every dependency preinstalled. Just keep the image tag in lockstep with the `playwright` version in `package.json` -- a mismatch means re-downloading browsers at container start, which is exactly the slow path you used the image to avoid.

**At scale:** Browser drift is the platform team's quietest failure mode: a runner built last month and a runner built today carry different engine builds, and a rendering-sensitive test passes on one and fails on the other. Pin one Playwright image tag in the shared pipeline template, upgrade it for all projects at once, and treat that upgrade like any dependency bump -- run the nightly matrix on it before rolling it out. Twenty projects on one pinned image is one upgrade to verify; twenty projects each resolving `latest` on their own schedule is twenty surprises.

---

## CI/CD Multi-Browser Strategy

```
┌──────────────────────────────────────────────────────────────────┐
│ Recommended CI/CD Strategy                                       │
│                                                                  │
│ Pull Request Checks:                                             │
│   Run on Chromium only (fast feedback)                           │
│   BROWSER=chromium npx cucumber-js --tags "@smoke"               │
│                                                                  │
│ Nightly Build:                                                   │
│   Run full suite on all browsers                                 │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐                 │
│   │ Chromium   │ │ Firefox    │ │ WebKit     │  (parallel)     │
│   │ Full suite │ │ Full suite │ │ Full suite │                 │
│   └────────────┘ └────────────┘ └────────────┘                 │
│                                                                  │
│ Release Pipeline:                                                │
│   Run full suite on all browsers (must pass to deploy)           │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐                 │
│   │ Chromium   │ │ Firefox    │ │ WebKit     │  (parallel)     │
│   │ All tests  │ │ All tests  │ │ All tests  │                 │
│   └────────────┘ └────────────┘ └────────────┘                 │
│         │               │              │                        │
│         └───────────────┼──────────────┘                        │
│                         ▼                                        │
│                   All pass? → Deploy                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

This shape is a budget decision as much as a quality one. Running three engines on every pull request would triple feedback time to catch compatibility bugs that almost never hide in a one-line change -- so PRs get Chromium speed, and the nightly matrix gets engine breadth while nobody is waiting on it. Wire each nightly lane to stamp its engine into the report metadata (`--browser firefox` on the report generator), and morning triage reads as one matrix instead of three disconnected reports.

---

## Browser Testing Quick Reference

```
┌───────────────────────────────────────────────────────────────────┐
│ What You Want                    │ Command                        │
├───────────────────────────────────────────────────────────────────┤
│ Run on Chromium (default)        │ npx cucumber-js                │
│ Run on Firefox                   │ BROWSER=firefox npx cucumber-js│
│ Run on WebKit                    │ BROWSER=webkit npx cucumber-js │
│ Run headed (visible browser)     │ HEADLESS=false npx cucumber-js │
│ Run with slow motion             │ SLOW_MO=1000 npx cucumber-js  │
│ Run fast (no delay)              │ SLOW_MO=0 npx cucumber-js     │
│ Install browser binaries         │ npx playwright install         │
│ Install with system deps         │ npx playwright install         │
│                                  │   --with-deps                  │
│ Run all browsers                 │ npm run test:all-browsers      │
│ Skip tests for a browser         │ --tags "not @skip-webkit"      │
└───────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- Playwright supports three browser engines: Chromium, Firefox, WebKit
- These three engines cover virtually 100% of real-world browsers
- Switch browsers with `BROWSER=chromium|firefox|webkit` environment variable
- Default is Chromium if `BROWSER` is not set
- Chromium gets special arguments (no-sandbox, ignore cert errors)
- Run headed mode with `HEADLESS=false` for debugging
- Control speed with `SLOW_MO` (0=fastest, 2000=slowest)
- Use browser-specific tags to skip known browser issues
- CI strategy: Chromium for PRs, all browsers for nightly/release
- Mobile emulation uses Playwright's built-in device profiles
- Install browsers with `npx playwright install`
- Your Gherkin features do not change between browsers -- the same tests run everywhere

Next: Responsive and Mobile Testing -- viewport control, breakpoints, and device emulation.

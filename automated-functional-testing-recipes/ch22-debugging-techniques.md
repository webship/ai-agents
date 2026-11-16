# Chapter 22: Debugging Techniques

---

A test fails. The error message says `Expected to find "Dashboard" but it was not present`. That tells you what went wrong but not why. Was the page still loading? Did the login fail? Was there a server error? Is the selector wrong?

This chapter arms you with every debugging technique available in webship-js, Playwright, and Cucumber-js. One theme runs through all of them: on a team that owns many suites, the person debugging a failure is usually not the person who wrote the scenario. So every technique here is also a way of attaching evidence to a failure -- screenshots, video, console errors, network logs -- until a red build on a project you have never opened arrives with everything you need to triage it.

---

## Recipe 204: Reading Cucumber Error Output

**Problem:** A red build lands in the team channel at 9 a.m. The reviewer on duty -- who owns fifteen other suites and did not write this one -- has thirty seconds to decide: real regression, environment hiccup, or broken test? The error text is the only evidence on the table, and most people skim past the one line that classifies the failure. Learning to read these messages is the fastest debugging skill you can build, because it tells you which of the other recipes in this chapter to reach for.

**Common error patterns:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Error Message                      │ Likely Cause                 │
├──────────────────────────────────────────────────────────────────┤
│ "Expected to find text X"          │ Page content not loaded yet  │
│                                    │ or wrong page displayed      │
│                                    │                              │
│ "Timeout 45000ms exceeded"         │ Whole step hit the Cucumber  │
│                                    │ step timeout: page never     │
│                                    │ loaded, or element never     │
│                                    │ appeared                     │
│                                    │                              │
│ "Element not visible"              │ Element exists but hidden    │
│                                    │ (display:none, behind modal) │
│                                    │                              │
│ "Strict mode violation"            │ Multiple elements match,     │
│                                    │ Playwright expects one       │
│                                    │                              │
│ "No visible modal found"           │ Modal did not open, or       │
│                                    │ wrong selector               │
│                                    │                              │
│ "Step undefined"                   │ No matching step definition  │
│                                    │ (check spelling, quotes)     │
│                                    │                              │
│ "ECONNREFUSED"                     │ Application is not running   │
│                                    │ at the expected URL          │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** the message encodes which layer gave up. `ECONNREFUSED` is the network layer -- the app is not running at `LAUNCH_URL`, so nothing browser-related matters yet. "Step undefined" is Cucumber's matcher -- your Gherkin text did not match any of the 411 built-in step expressions, usually a typo or a missing quote (Recipe 205 catches these in seconds). "Timeout 45000ms exceeded" is the Cucumber step timeout from the default profile -- the step as a whole never finished, which usually means a navigation hung. A web-first assertion failure ("expected ... to be visible") means the framework already did the patient part for you: it polled every 100ms for the full 5-second window, through smartSettle's AJAX/timer/DOM-quiet checks, and the element genuinely never reached that state. That last point changes your diagnosis: with webship-js 2.0 you can mostly stop asking "did the test wait long enough?" and start asking "why did the app never produce this?" -- a much better question.

**At scale:** write this table into a one-page triage guide and put it in every project repository. The goal is that anyone on the team -- including the engineer seeing this project for the first time -- can classify a failure (app down / test broken / regression / flake) without summoning the person who wrote the suite. Cross-project on-call rotation only works when the failure taxonomy is shared.

---

## Recipe 205: Using Dry-Run to Verify Step Matching

**Problem:** A product owner drafted ten new scenarios straight from the acceptance criteria -- exactly what you want to encourage. But one step says `When I fill "Email" with` instead of `When I fill in "Email" with`, and you only find out twelve minutes into the run when the scenario dies as UNDEFINED. Worse: the failure makes the PO feel they "broke the tests", which is the fastest way to make them stop writing scenarios.

**Solution:**

```bash
npx cucumber-js --dry-run

# Output shows each step's match status:
# ✓ Given I am on homepage             → matched
# ✓ When I go to "/login"              → matched
# ? When I do something custom          → UNDEFINED (no match!)
```

This catches typos and missing step definitions before you waste time running tests.

**How It Works:** `--dry-run` loads everything a real run loads -- the feature files, the 411 built-in step definitions, your custom steps from `tests/step-definitions/` -- and runs the matching phase only. Every step's text is checked against the registered Cucumber expressions; no browser launches, no page loads, so a whole suite verifies in a couple of seconds. It catches the silent killers: a missing `in`, curly quotes pasted from a Word document, a custom step that exists on one project but not the one you copied the scenario into. It cannot catch wrong selectors or wrong expectations -- only that every line of Gherkin has an implementation waiting for it.

**At scale:** make `--dry-run` the first job in every project's pipeline -- it costs seconds and fails fast, before you pay for browser minutes. It is also your portability check: before promoting a feature file from one project into a shared library, dry-run it against each consuming project to prove every step resolves there too.

---

## Recipe 206: Running Tests in Headed Mode

**Problem:** The error says the dashboard never appeared. You have re-read the scenario five times and it looks right. You are now debugging blind -- reasoning about what a headless browser might have shown, instead of just looking. Ten minutes of guessing could have been thirty seconds of watching.

**Solution:**

```bash
# Watch the test run in a real browser window
HEADLESS=false npx cucumber-js tests/features/login.feature

# Slow it down so you can follow
HEADLESS=false SLOW_MO=2000 npx cucumber-js tests/features/login.feature
```

This is the single most effective debugging technique. You can see:
- Whether the page loaded
- Whether the login form appeared
- Whether the button was clicked
- What error messages the UI shows

**How It Works:** `HEADLESS=false` flips the Playwright launch option in `playwright.config.ts` -- the exact same Chromium (or Firefox/WebKit via the `BROWSER` env var) runs, just with a window. webship-js also adapts its pacing: `SLOW_MO` defaults to 0 in CI, 300ms in headless local runs, and 800ms when headed, so a headed run is already watchable without any flags; set `SLOW_MO=2000` when you need to read each action. Nine times out of ten the bug is obvious within one viewing -- a validation error under the form, a cookie-consent banner covering the button, a redirect to the wrong environment. The failure modes you cannot see in an error message are exactly the ones you can see through a window.

**Pro Tip:** combine headed mode with a single feature file path, as above. Watching one failing scenario is debugging; watching the whole suite is a movie you do not have time for.

**At scale:** headed mode is the one technique in this chapter that does not travel to CI -- there is no window on a runner. The multi-project habit that replaces it: reproduce locally against the failing project's own environment by pointing `LAUNCH_URL` at it, set `BROWSER` to whatever the pipeline used, and watch the exact scenario that failed. Most "CI-only" failures stop being CI-only the moment the same scenario runs against the same environment with your eyes on it -- and the ones that survive that test are genuinely about the runner, which is your platform team's lead, not yours.

---

## Recipe 207: Taking Screenshots Mid-Test

**Problem:** The failure only happens in CI, where there is no window to watch. The After-failure screenshot shows you the crime scene, but not the lead-up -- was the form ever filled? Did the page render before the click? You need the browser's state at specific moments along the way, not just at the end.

**Solution:**

```gherkin
Feature: Debug Login
  Scenario: Debug login failure
    Given I am on homepage
    When I go to "/login"
    And I take a screenshot named "01-login-page"
    And I fill in "Email" with "admin@example.com"
    And I fill in "Password" with "AdminPass123"
    And I take a screenshot named "02-form-filled"
    And I press "Sign In"
    And I take a screenshot named "03-after-login"
    Then I should see "Dashboard"
```

The screenshots create a visual timeline of the test execution. Examine them in `screenshots/` to see exactly what the browser showed at each point.

**How It Works:** you rarely need to hand-place screenshot steps, because webship-js captures automatically. `WEBSHIP_SCREENSHOT_ON_FAILED` is `true` by default -- every failing scenario already leaves a `failed_`-prefixed image in `WEBSHIP_SCREENSHOT_DIR` (default `./screenshots`). For a full timeline, tag the scenario `@screenshots` and the framework captures after *every* step of that scenario -- the tag version of `WEBSHIP_SCREENSHOT_ON_EVERY_STEP`, scoped to just the scenario you are investigating. File names follow `WEBSHIP_SCREENSHOT_PATTERN` (`{datetime}.{feature_file}.feature_{step_line}.{ext}` by default), so each image points back to the exact step line that produced it. The named-screenshot steps in the example are still the right tool when you want a stable, human-readable name to compare run over run.

**At scale:** keep the naming pattern identical across all projects and have CI archive the screenshots directory as a build artifact on failure. When a remote engineer triages another team's red build, "open artifacts, read filenames in order, find the step line" is a procedure that works on every project the same way -- no tribal knowledge needed. Set `WEBSHIP_SCREENSHOT_PURGE` so a runner that survives between builds does not mix last night's images into today's investigation.

---

## Recipe 208: Browser Console Log Capture

**Problem:** The button click does nothing, the assertion fails, and the screenshot shows a perfectly normal-looking page. What it does not show: the JavaScript exception that killed the click handler. In a headless CI run, console errors vanish into nowhere -- the most informative line of evidence in the whole failure, and nobody saw it.

**Solution:** webship-js 2.0 captures page JavaScript errors out of the box -- you configure it, not code it:

```bash
# Default: errors are reported as warnings alongside the scenario
WEBSHIP_JS_ERROR_MODE=warn npx cucumber-js

# Strict: any page JS error fails the scenario
WEBSHIP_JS_ERROR_MODE=fail npx cucumber-js

# Widen capture to warnings and info, ignore known third-party noise
WEBSHIP_JS_ERROR_LEVELS=error,warning \
WEBSHIP_JS_ERROR_IGNORE="analytics|gtag" \
npx cucumber-js
```

Per-scenario control via tags: `@js-fail` (or `@javascript`) makes errors fail that scenario, `@js-warn` downgrades to warnings, `@js-off` disables capture entirely.

If you also want non-error console output (logs, debug noise) attached to failures, add a custom hook:

```javascript
// tests/support/hooks.js
const { Before, After } = require('@cucumber/cucumber');

Before(async function () {
  // After opening browser, set up console log capture
  this.consoleLogs = [];
  this.page.on('console', msg => {
    this.consoleLogs.push({
      type: msg.type(),
      text: msg.text(),
    });
  });
});

After(async function (scenario) {
  // Print console errors if test failed
  if (scenario.result.status === 'FAILED') {
    const errors = this.consoleLogs.filter(l => l.type === 'error');
    if (errors.length > 0) {
      console.log('\n--- Browser Console Errors ---');
      errors.forEach(e => console.log(`  [${e.type}] ${e.text}`));
      console.log('---');
    }
  }
});
```

**How It Works:** the built-in capture listens for page errors on every scenario and routes them according to `WEBSHIP_JS_ERROR_MODE` -- `warn` (the default) surfaces them in the output without failing anything, `fail` turns the page's JavaScript health into an assertion, `off` silences it. `WEBSHIP_JS_ERROR_LEVELS` controls which console levels count (just `error` by default), and `WEBSHIP_JS_ERROR_IGNORE` takes a regex for the inevitable third-party noise -- analytics beacons, ad scripts, browser-extension chatter -- so one flaky CDN does not fail your suite. The tag overrides matter for precision: keep the suite in `warn` globally, then mark your most critical journeys `@js-fail` so a broken click handler on checkout is a failure, not a footnote.

**At scale:** standardize the env vars in your shared CI templates instead of per-project configs -- one place to tighten `warn` to `fail` when a portfolio is ready, one shared `WEBSHIP_JS_ERROR_IGNORE` pattern for the marketing scripts every site carries. JS-error capture is also a free early-warning system: a new error appearing across several projects' runs on the same night usually means a shared dependency or CDN changed, and you have found it before any user reported it.

---

## Recipe 209: Network Request Inspection

**Problem:** The UI swallows failures politely: the login form just sits there, no error toast, nothing in the screenshot. Meanwhile the API returned a 401 that the front end never surfaced. Without seeing the traffic, you will debug the form for an hour when the real story is one HTTP status code away.

**Solution:** for assertions inside a scenario, use the built-in recording steps:

```gherkin
Given I start recording network requests
When I press "Sign In"
Then a POST request to "**/api/login" should have been made
```

For automatic evidence on every failure, add a hook that logs error responses:

```javascript
// tests/support/hooks.js
Before(async function () {
  this.networkRequests = [];
  this.page.on('response', response => {
    if (response.status() >= 400) {
      this.networkRequests.push({
        url: response.url(),
        status: response.status(),
        method: response.request().method(),
      });
    }
  });
});

After(async function (scenario) {
  if (scenario.result.status === 'FAILED' && this.networkRequests.length > 0) {
    console.log('\n--- Failed Network Requests ---');
    this.networkRequests.forEach(r =>
      console.log(`  ${r.method} ${r.url} → ${r.status}`)
    );
    console.log('---');
  }
});
```

Output on failure:

```
--- Failed Network Requests ---
  POST http://localhost:8080/api/login → 401
  GET http://localhost:8080/api/dashboard → 403
---
```

That output reads like a confession: the login API rejected the credentials (401), so the dashboard refused access (403). The form was never the problem.

**How It Works:** the built-in steps and the custom hook tap the same source -- Playwright's network events on the page. `Given I start recording network requests` begins collecting every request the browser makes; `Then a POST request to "**/api/login" should have been made` matches the log against a glob pattern, the same pattern syntax `page.route()` uses for mocking. That makes it the assertion of choice for invisible behavior: analytics calls, autosaves, background syncs -- things with no UI to assert on. The hook variant filters `response` events for status >= 400 and prints them only when the scenario fails, so passing runs stay quiet and failing runs carry their own network evidence.

**At scale:** ship the failed-request hook in your shared support package so every project's failure output includes it -- it converts a whole class of cross-team tickets from "the test is flaky" to "your API returned 500 at 02:14, here is the URL". DevOps will notice the difference: failures that arrive with status codes and endpoints get routed to the right owner on first triage instead of bouncing through the QA channel.

---

## Recipe 210: Video Recording of Test Runs

**Problem:** A scenario fails one night in five, only in CI, never locally. The screenshot shows the end state; what you need is the thirty seconds *before* -- the redirect that happened, the spinner that never resolved, the modal that flashed and vanished. For intermittent CI failures, video is the evidence that ends the speculation.

**Solution:** video recording is built into webship-js 2.0 -- one environment variable, no hook code:

```bash
# Record only scenarios that fail (the right default for CI)
WEBSHIP_VIDEO=on-failure npx cucumber-js

# Record everything (debugging sessions)
WEBSHIP_VIDEO=on npx cucumber-js

# Record only scenarios tagged @video
WEBSHIP_VIDEO=tag npx cucumber-js

# Choose where videos land (default ./videos)
WEBSHIP_VIDEO=on-failure WEBSHIP_VIDEO_DIR=./artifacts/videos npx cucumber-js
```

Per-scenario control in the feature file:

```gherkin
@video
Scenario: Checkout completes with a saved card
  # Always recorded, regardless of pass/fail

@no-video
Scenario: Bulk import of 5000 rows
  # Never recorded -- long scenario, huge file, no debugging value
```

**How It Works:** under the hood webship-js passes Playwright's `recordVideo` context option for you -- 1280x720 WebM files, named by the pattern `{datetime}.{feature_file}.{scenario}.{status}.{ext}` so a directory listing already tells you which scenario failed and when. Playwright can only decide to record when the browser context is created, which is why this is a context-level setting and why `on-failure` mode records everything and discards the passes. It is also why the mid-scenario `start/stop video recording` steps come with a warning: starting recording mid-flight forces webship-js to rebuild the browser context, and the page state -- cookies, session, the page you were on -- is lost. Use the env var or tags instead unless you genuinely need a partial recording and your scenario re-establishes its own state afterward.

**At scale:** standardize on `WEBSHIP_VIDEO=on-failure` in the shared CI template and have the pipeline upload `WEBSHIP_VIDEO_DIR` as an artifact only when the job fails. Green runs cost you nothing; red runs come with a replay any engineer on any team can watch. Storage adds up across forty nightly suites, so set an artifact retention window -- a video older than the next deploy has already done its job.

---

## Debugging Quick Reference

```
┌───────────────────────────────────────────────────────────────────┐
│ Technique                        │ When to Use                    │
├───────────────────────────────────────────────────────────────────┤
│ HEADLESS=false                   │ First thing to try             │
│ SLOW_MO=2000                     │ Test runs too fast to follow   │
│ --dry-run                        │ Step not matching              │
│ @screenshots tag                 │ Visual timeline of a scenario  │
│ WEBSHIP_JS_ERROR_MODE=warn/fail  │ Suspect JS errors              │
│ Network request logging          │ Suspect API failures           │
│ WEBSHIP_VIDEO=on-failure         │ Intermittent failures in CI    │
│ print API response               │ API test returns wrong data    │
│ print current URL                │ Lost in a redirect chain       │
│ Screenshot on failure            │ On by default -- keep it on    │
└───────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- Read error messages carefully -- they indicate which layer gave up, and which recipe to reach for
- `--dry-run` catches undefined steps in seconds, before any browser launches
- `HEADLESS=false` is the most effective debugging technique; `SLOW_MO` makes it watchable
- The `@screenshots` tag turns one scenario into a visual timeline; failure screenshots are on by default
- `WEBSHIP_JS_ERROR_MODE` captures page JavaScript errors -- `warn` globally, `@js-fail` on critical journeys
- Built-in network recording steps plus a failed-request hook turn API failures into evidence
- `WEBSHIP_VIDEO=on-failure` is the right CI default for intermittent failures
- Standardize all of it in shared CI templates so every project fails with the same evidence attached

Next: Test Maintenance -- keeping your test suite healthy over time.

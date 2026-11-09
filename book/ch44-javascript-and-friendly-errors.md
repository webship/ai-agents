# Chapter 44: JavaScript Errors and Tester-Friendly Error Handling

---

A passing test that leaves a stack trace in the browser console is a lie. Functionally the user got through. But the next user on a slightly different browser version, or with a malformed cookie, or after a slow CDN response will hit that latent error and break. Webship-js ~2.0 added a **JavaScript errors and warnings** family that turns every test into a smoke test for console health. It also added a **tester-friendly error system** that swaps Playwright's raw exception output for messages a non-engineer can read.

This chapter covers both: how to assert console health as part of every scenario, and how the friendly error pipeline reshapes failures into something a test author can act on without a debugger.

---

## Why Console Health Matters

<!-- diagram: page-lifecycle -->

Browsers run JavaScript permissively. An uncaught exception in a third-party widget, a 404 from a lazy-loaded script, a deprecated API warning -- none of these stop the page. They quietly log to the console. By the time a user reports "the page feels broken", the actual cause is buried in three weeks of console noise.

A test that runs every commit can catch these the day they appear -- but only if the test is wired to listen. The webship-js JavaScript-errors family installs a listener on `page.on('pageerror')` (uncaught exceptions) and `page.on('console')` (console output) for every scenario. Every event is appended to a per-scenario list in the order it fired, each entry carrying its capture type (`pageerror`, `console.error`, `console.warning`, ...) plus the message text. That list is what every assertion in this chapter reads.

The whole family is controlled by a small set of environment switches, so the capture policy lives in your CI configuration instead of your feature files:

- `WEBSHIP_JS_ERROR_MODE` -- `warn` (default: log to the report, don't fail), `fail` (any captured error fails the scenario), or `off`
- `WEBSHIP_JS_ERROR_LEVELS` -- which console levels count; default `error`, add `warning,info` to widen the net
- `WEBSHIP_JS_ERROR_IGNORE` -- a regex of known noise; matching entries are filtered out before the error checks evaluate the list
- `WEBSHIP_JS_ERROR_AFTER` (default on; set `0` to disable) -- run the automatic check at the end of every scenario
- `WEBSHIP_JS_ERROR_BEFORE` (off by default; set `1`) -- also log a pre-scenario snapshot, useful when a page survives across scenarios

Per-scenario tag overrides beat the environment: `@js-fail` (or `@javascript`) forces fail mode, `@js-warn` downgrades to warn, `@js-off` disables capture for that scenario. Keep this control panel in mind -- the recipes below lean on it constantly.

---

## Recipe 429: Asserting No JavaScript Errors

### Problem
The Thursday before release, the smoke suite is green and everyone signs off. What nobody saw: the landing page has been throwing `TypeError: Cannot read property 'x' of undefined` since Monday's deploy. The assertion checked the visible headline, the headline was there, and nobody asked the console. Two weeks later the same latent error starts swallowing click handlers on a browser version you never tested, and the bug report says "the page feels broken" with zero clues.

### Solution

```gherkin
Feature: Console health on landing page

  Scenario: Landing page is console-clean
    Given I am on "/"
    Then there should be no JavaScript errors
```

Run this at the end of every smoke scenario. The step is a hard assertion over the per-scenario error list that the `page.on('pageerror')` listener has been filling since the page was created. If any uncaught exception fired at any point during the scenario -- not just on the final page -- the step fails and the full error text lands in the report as a numbered list, so the person reading the failure sees the actual `TypeError`, not a vague "console check failed."

Two internals worth knowing. First, this step is a hard check: it fails on captured errors even when the suite runs in the default `warn` mode. Second, it marks the list as asserted, so the automatic end-of-scenario check skips -- one error is never reported twice, once by your step and once by the framework.

Note what you get even without this step: the default capture mode is `WEBSHIP_JS_ERROR_MODE=warn`, so captured errors already show up in the report as warnings. The explicit step turns that passive signal into a gate for the scenarios where console health is part of the contract.

**At scale:** across 20 projects, don't paste this step into every feature file. Set `WEBSHIP_JS_ERROR_MODE=fail` once in the CI environment and let the automatic after-scenario check do the work everywhere. Exceptions then become visible tags (`@js-warn`, `@js-off`) instead of silently missing steps -- and a reviewer can spot a relaxed scenario in the diff.

---

## Recipe 430: Asserting No JavaScript Warnings

### Problem
Your pages throw zero errors, so the team stopped looking at the console months ago. Meanwhile it has quietly filled with `[Deprecation] Synchronous XMLHttpRequest on the main thread is deprecated` -- a countdown timer, not a footnote. The day Chrome actually removes that API, the feature breaks in production and the fix becomes an emergency patch instead of a planned ticket. You want a guard that notices the countdown while it is still cheap to act on.

### Solution

```gherkin
  Scenario: Landing page is warning-clean
    Given I am on "/"
    Then there should be no JavaScript warnings
```

Warnings arrive through the same `page.on('console')` listener as errors, just at a different level -- and here is the catch: the level filter runs at capture time. With the default `WEBSHIP_JS_ERROR_LEVELS=error`, warning-level messages are never recorded at all, and this step would pass on a page drowning in deprecations. Pair it with `WEBSHIP_JS_ERROR_LEVELS=error,warning` so warnings actually enter the list. The step then filters the captured entries for `console.warning` and fails if any survived -- which lets you keep the suite-wide policy strict about errors and use targeted scenarios like this one to police warnings where they matter.

Use warnings as a smoke check for technical debt. Deprecation warnings predict future breakage -- catching them before the browser removes the deprecated API saves an emergency patch.

**At scale:** deprecations hurt worst at fleet scale. If 30 of your sites share the same base theme or framework version, the day a browser removes a deprecated API, all 30 break the same week. One warning-clean sentinel scenario per project, or `WEBSHIP_JS_ERROR_LEVELS=error,warning` on a nightly profile, turns that cliff into a backlog item you saw coming a year out.

---

## Recipe 431: Matching Errors Against a Pattern

### Problem
The third project you onboard arrives with a tag manager, two ad networks, and a chat widget -- each of them throwing errors you will never be allowed to fix. Turn on strict console checking and every scenario fails on someone else's code; turn it off and your own bugs sail through with the noise. You need a middle path: stay silent about the rented scripts, fail loudly on anything from your own namespace.

### Solution

```gherkin
  Scenario: No errors from our own code
    Given I am on "/"
    Then JavaScript errors should not match "myapp\\.|MyApp"
```

The pattern is a JavaScript regex tested against the text of every recorded error. The step fails if any of them matches -- so here, third-party noise is invisible to the assertion while anything mentioning `myapp.` or `MyApp` trips it.

There is a second tool for the same problem, working from the opposite end: `WEBSHIP_JS_ERROR_IGNORE` takes a regex of noise to discard -- matching entries are filtered out before `there should be no JavaScript errors` and the automatic end-of-scenario check evaluate the list. Use `WEBSHIP_JS_ERROR_IGNORE` for permanent, suite-wide noise (the ad network that ships with every page) and the in-scenario negative match for per-scenario gating on first-party errors. One deliberate asymmetry: `print JavaScript errors` and the pattern step in this recipe read the raw, unfiltered list. Diagnostics stay honest, and a targeted match can still catch an error the suite has muted.

**At scale:** keep one `WEBSHIP_JS_ERROR_IGNORE` value in your shared CI configuration, not a per-project copy. When the team agrees that a vendor error is officially "not ours", it gets added once, with a comment and a ticket link, and every project inherits the decision. The day a project needs its own exception, that diff is one line in one place -- easy to review, easy to revert when the vendor finally fixes it.

---

## Recipe 432: Printing Errors for Debugging

### Problem
There is a test that fails roughly once a week, always with the same unhelpful timeout, never on anyone's laptop. You have stared at five failure screenshots and learned nothing -- the page looks fine in all of them. Your hunch is a JavaScript error that only fires under CI timing, but by the time you open the report, the console evidence is gone. You want every run to log its full console error list so the next failure carries its own forensics.

### Solution

```gherkin
  Scenario: Debug captures console errors
    Given I am on "/checkout"
    When I click "Place Order"
    Then ".confirmation" should be visible within 10 seconds
    And print JavaScript errors
```

`print JavaScript errors` writes the captured list to the test output, and from there into the JSON report (`tests/reports/cucumber_report.json`), where it survives as long as your CI keeps the artefact. Each entry carries the error text plus source location, so when the weekly flake finally fires again you can diff this run's console against last week's instead of guessing.

The step is deliberately a no-op assertion -- it never fails. That makes it safe to leave in place on suspect scenarios while you collect evidence across runs. Once you have identified the culprit error, replace the print with a real assertion (`JavaScript errors should not match "..."`) and the diagnosis becomes a regression guard.

---

## Recipe 433: Combining Console Checks With Web-First Matchers

### Problem
The contact form works -- for now. The success banner appears, the test goes green, and a `TypeError` fires silently in the analytics callback right after submit. Nobody notices until the analytics vendor updates their script and that swallowed error starts blocking the redirect. The scenario asserted what the user sees and ignored what the browser knows. Test both.

### Solution

```gherkin
  Scenario: Submit succeeds AND console is clean
    Given I am on "/contact"
    When I fill in "name" with "Maria"
    And I fill in "email" with "maria@example.com"
    And I press "Send"
    Then ".success-banner" should be visible within 10 seconds
    And there should be no JavaScript errors
```

Two contracts, one scenario. The ordering here is doing quiet work. The web-first visibility check polls every 100ms for up to 10 seconds, so by the time it passes, the submit round-trip is done. Then webship-js's after-step settle kicks in -- after every state-changing step, an AfterStep hook runs `smartSettle(page, 1500)`, which waits until `window.__webshipAjaxCount` is zero, pending timers are zero, and the DOM has been quiet for 250ms. That means a late error thrown from a fetch callback or a `setTimeout` has already fired and been captured by the listener before the final console assertion reads the list. You are not racing the page; the framework already waited for it.

Avoid the trap of asserting only the visible outcome -- a green page with a hidden error is a latent bug with your name on the commit.

---

## The Friendly Error System

Playwright's default error output is precise but mechanical:

```
Error: locator.click: Timeout 30000ms exceeded.
=========================== logs ===========================
waiting for locator('text=Save')
============================================================
    at /tests/step-definitions/action.steps.js:42:18
```

For an engineer, that's enough. For a tester, a product manager, or a junior team member, it's a wall. They see a stack trace and email an engineer. Multiply that across a team running dozens of suites and the failure-triage queue becomes one person's full-time job -- not because the failures are hard, but because only one person can read them.

Webship-js ~2.0 ships a **`friendly()` and `humanize()` pipeline** that wraps every step's error in a structured, human-readable form. The same failure now reads:

```
Step failed: When I press "Save"

Could not find a button or link labelled "Save" within 30 seconds.

Hints:
  - The label may have changed. Try "Save changes" or "Submit".
  - The element may not be visible. Did you wait for the form to load?
  - The element may be inside an iframe. Did you switch to it first?

Original error: locator.click: Timeout 30000ms exceeded.
```

The tester sees the actionable summary. The original error stays at the bottom for engineers.

---

## Recipe 434: Friendly Error in Action

### Problem
A step `When I press "Save"` fails on the new hire's first PR. They open the report, find a 30-second-timeout stack trace pointing into `action.steps.js`, and conclude the framework is broken -- when the actual story is that a designer renamed the button to "Save changes". You want the report to tell that story in plain English, to whoever is reading it.

### Solution
You don't write anything. Webship-js wraps the step automatically. The report shows the friendly summary plus the original error. Custom steps you write benefit from the same pipeline if you wrap your error throws with `friendly()`:

```js
const { friendly } = require('./webship');

When('I confirm the order in {string}', async function(button) {
  const locator = this.page.getByRole('button', { name: button });
  if (!(await locator.isVisible())) {
    throw friendly(`Could not find a confirmation button labelled "${button}". The order may not have reached the confirmation step yet.`);
  }
  await locator.click();
});
```

`friendly()` takes three forms. `friendly(message)` makes your sentence the entire failure output -- it sets the Error's `.stack` to the rendered message, which is the trick that stops cucumber-js printing a JavaScript stack trace at a tester. `friendly(message, err)` appends a `Why:` line with the original error run through `humanize()` (more on that in the next recipe). And `friendly({ action, target, hint, cause })` builds the structured form the built-in steps use: "Could not *action* '*target*'", a humanized `Why:`, and your `Hint:` line. Reach for the object form when you have a hint to offer -- that is where the hints in the report come from.

The message you write matters more than the mechanism. "Could not find a confirmation button" is good; "The order may not have reached the confirmation step yet" is better, because it points at the most likely cause instead of just restating the symptom. Write the hint you would give a colleague who walked over to your desk.

---

## Recipe 435: Humanizing Selector Errors

### Problem
A custom step throws with a raw CSS selector in the message: `Could not find #cta-primary--btn`. The tester reading the report has never seen that selector -- it lives three layers down in a step-definition file they have no reason to open. So the failure that should have taken thirty seconds to triage ("oh, the CTA button moved") instead becomes a Slack thread, a screen share, and an engineer's afternoon.

### Solution
Two moves: name the element in human terms yourself, and let `humanize()` translate the low-level cause:

```js
const { friendly, humanize } = require('./webship');

try {
  await locator.click();
} catch (e) {
  throw friendly({
    action: 'click',
    target: 'the CTA button',
    cause: e,
    hint: 'the button may have been renamed or moved below the fold.',
  });
}
```

`humanize()` is a translator for raw error text, not a selector lookup. It takes the original Playwright or Node error and maps known patterns to one plain-English sentence: `Timeout 30000ms exceeded` becomes "the page took too long to respond", `strict mode violation` becomes "the locator matched more than one element", `ECONNREFUSED` becomes "the server is not running or refused the connection". Around twenty patterns ship with webship-js; anything unmatched falls back to a cleaned-up first line with Playwright's noisy `Call log:` suffix stripped. Inside the object form of `friendly()`, the `cause` you pass is rendered through it as the `Why:` line, so the failure above reads: `Could not click "the CTA button". Why: the element could not be reached. Hint: the button may have been renamed or moved below the fold.`

The human name for the element is on you -- and the natural source is the names you already maintain in the `tests/selectors/` JSON files for named-selector steps. If the registry calls `#cta-primary--btn` "cta button", say "cta button" in your `target` too. One vocabulary for steps and for failures.

**At scale:** this is one of the strongest arguments for shared selector JSON files across projects. When 20 sites built on the same platform share a `common-selectors.json`, a failure on any of them reads the same way -- "Could not find 'main menu'" means the same thing on site 3 and site 17. A reviewer who triages failures across the whole portfolio never has to learn per-project selector dialects.

---

## Recipe 436: Per-Scenario Hooks for JavaScript Mode

### Problem
Your staging suite should scream on every console error -- you own that code and you can fix it the same day. But the same suite also runs as a smoke check against production, where a marketing tag you are not allowed to remove throws on every page load. One global setting cannot serve both. The first time this hurts is the morning the production smoke run goes red over the marketing tag and someone pages the on-call engineer for a bug that is three months old and not yours.

### Solution
The mode is environment-driven, with tag overrides per scenario -- no custom hook required. In each pipeline, set the mode that matches what you can act on:

```bash
# staging pipeline: any console error fails the scenario
WEBSHIP_JS_ERROR_MODE=fail npm test

# production smoke pipeline: capture off entirely
WEBSHIP_JS_ERROR_MODE=off npm test
```

For scenario-level exceptions, use the built-in tags:

```gherkin
  @js-off
  Scenario: Legacy page we cannot change
    Given I am on "/legacy-widget"
    Then ".widget" should be visible

  @js-fail
  Scenario: Our own checkout must be console-clean
    Given I am on "/checkout"
    Then ".cart-summary" should be visible
```

Modes:

- `warn` -- log captured errors to the report but don't fail (the default)
- `fail` -- any captured error fails the scenario
- `off` -- don't capture at all

The resolution order is fixed and worth memorising: scenario tag beats env var beats `worldParameters.javascript.mode` in `cucumber.js` beats the built-in default of `warn`. So `@js-fail` (or the legacy `@javascript`) forces fail mode for one scenario, `@js-warn` downgrades to warn, `@js-off` (or legacy `@js-errors`) disables capture -- regardless of what the pipeline exported. The tags are implemented as a Before hook inside webship-js, so you get per-scenario hook behaviour without writing one. Two companion switches shape the automatic check itself: `WEBSHIP_JS_ERROR_AFTER` (on by default; set `0` to skip the end-of-scenario evaluation) and `WEBSHIP_JS_ERROR_BEFORE` (set `1` to also log a pre-scenario snapshot -- usually empty, but a lifesaver when scenarios share a long-lived page).

**At scale:** the env-var-plus-tags split is exactly what a multi-project CI farm wants. The farm sets `WEBSHIP_JS_ERROR_MODE=fail` once, in shared pipeline configuration; individual projects never edit it. Exceptions live as tags inside feature files, where they show up in pull-request diffs and someone has to defend them in review. The day you wonder "which scenarios have we exempted across the portfolio?", the answer is one grep for `@js-off`.

---

## Recipe 437: Filtering Stdout in Reports

### Problem
A delivery manager opens the test report before a release call, hoping for a yes/no answer. What they get is two hundred lines of Playwright internals, GPU warnings, and `Forced color profile` noise wrapped around the four lines that matter. They close the report and ask an engineer instead -- which means the report failed at its one job: letting a non-engineer read the result.

### Solution
The friendly error pipeline wraps `stdout` and `stderr` with a line filter that strips cucumber-js's hook-status noise -- the `✔ Before # ...` and `✔ After # ...` lines that webship-js's own hooks (browser setup, settle, screenshot, video, JS-error capture) would otherwise print around every scenario in a failure dump. You don't enable it; it's on by default. A suite this instrumented runs half a dozen hooks per scenario, and without the filter a single failing step arrives wrapped in a dozen lines of green checkmarks nobody asked about. Disable it with `WEBSHIP_FILTER_HOOK_LINES=off` when you suspect a hook itself is misbehaving and you want to see every one of them.

The filter only strips hook chatter. When you genuinely need the raw firehose locally -- usually when debugging a step definition, not a page -- ask Playwright for its own debug channel directly:

```bash
DEBUG=pw:api npm test
```

That prints every Playwright API call with timings to your terminal, unfiltered, while the report stays clean. Best of both: forensic detail where the engineer is looking, readable output where everyone else is.

---

## Recipe 438: Capturing the First Error

### Problem
A broken deploy takes out one JavaScript module, and suddenly the scenario reports fourteen errors. Thirteen of them are downstream casualties -- handlers that never registered, objects that never initialised. The person triaging reads all fourteen, gets overwhelmed, and files a bug describing error number nine. You want the failure message to lead with the one error that caused the rest.

### Solution
Use the helper:

```js
const errors = this._jsErrors || [];
if (errors.length) {
  throw friendly(`First JavaScript error: ${errors[0].message}`);
}
```

`this._jsErrors` is the per-scenario list the built-in steps read -- the listeners push entries the moment events fire, each shaped `{ type, message }`, so index 0 is the first error chronologically. Uncaught exceptions (`pageerror`) and console output interleave in real firing order, and in a cascade, the first entry is the root cause far more often than not. Everything after it tends to be the page failing to cope with the first failure.

This is the same triage rule compilers taught us decades ago: fix the first error, ignore the rest, recompile. Wrapping it in `friendly()` means the report leads with that one actionable line instead of a fourteen-item wall.

---

## Recipe 439: Asserting Specific Errors Are Allowed

### Problem
Your CI runners have no outbound internet, so every run logs `Failed to load Sentry beacon` -- an error that will never happen for a real user and that you cannot fix from the test side. The lazy move is to stop checking the console entirely, and that is how a real `TypeError` shipped unnoticed the last time someone made that call. You want a precise allowlist: that one error is fine, everything else still fails.

### Solution

```gherkin
  Scenario: Page errors except Sentry beacon
    Given I am on "/"
    Then JavaScript errors should not match "^(?!Failed to load Sentry beacon).*"
```

The pattern uses a negative lookahead anchored at `^`: any error that does **not** begin with the allowed text matches the pattern, and the step fails on a match. The anchor matters -- without `^`, the regex engine would retry the lookahead from position 1 onwards and end up matching even allowed errors.

For multiple allowed errors, list them with alternation:

```gherkin
Then JavaScript errors should not match "^(?!(Failed to load Sentry beacon|Cannot load Google Tag Manager)).*"
```

If an error should be allowed everywhere, not just in one scenario, promote it out of the feature file and into `WEBSHIP_JS_ERROR_IGNORE` -- the suite-wide regex filter. Matching entries are dropped before `there should be no JavaScript errors` and the automatic mode check evaluate the list, so the noise stops failing scenarios everywhere at once. (The raw list still keeps them, which is why `print JavaScript errors` continues to show them -- diagnostics stay unfiltered on purpose.) Scenario-level lookaheads are for local exceptions; the env var is for suite policy.

**At scale:** treat the ignore list like a dependency: versioned, commented, reviewed. One shared `WEBSHIP_JS_ERROR_IGNORE` value in the CI farm config, with each fragment annotated by ticket ("Sentry beacon -- runners are offline, INFRA-204"), means a new project inherits years of accumulated judgment on day one instead of rediscovering each noise source the hard way.

---

## Recipe 440: Asserting Console Cleanliness for the Whole Suite

### Problem
You have 400 scenarios across 30 feature files, and the team agrees console errors should fail tests. Adding `Then there should be no JavaScript errors` to 400 scenarios is a diff nobody wants to review, and the new hire's first feature file will forget the line anyway. Policy that depends on every author remembering a step is not policy -- it's hope.

### Solution
Add an After hook:

```js
const { After } = require('@cucumber/cucumber');
const assert = require('assert');

After({ tags: 'not @allow-console-errors' }, function() {
  const errors = this._jsErrors || [];
  if (errors.length) {
    throw new Error(`Console had ${errors.length} JavaScript error(s):\n${errors.map(e => e.message).join('\n')}`);
  }
});
```

Every scenario now gets the check for free. Tag exceptional scenarios with `@allow-console-errors` to opt out.

Before you write this hook, know that webship-js ships the same behaviour built in: `WEBSHIP_JS_ERROR_MODE=fail` with `WEBSHIP_JS_ERROR_AFTER` left at its default `true` evaluates the captured error list at the end of every scenario automatically, and `@js-off` is the built-in opt-out tag. Zero code, same guarantee. Hand-roll the After hook only when you want something the built-in doesn't give you -- a custom message format, a different tag name your suite already standardised on, or extra logic like posting the error list to a chat channel.

**At scale:** the built-in wins specifically because it's configuration, not code. A hook lives in one repo and gets copy-pasted (and then drifts) across the other 29; an env var lives in the shared CI template and applies to every project the moment it's set. When the policy changes -- say the team decides to downgrade to `warn` during a big migration -- that's a one-line change in one pipeline file, not 30 pull requests.

---

## Recipe 441: Asserting Page Errors on Navigation

### Problem
Some pages have a criminal record. `/admin/imports` lazy-loads a heavy bundle, and twice in the past year a deploy left its chunk manifest stale -- both times the bug report said "the imports page is blank", and both times triage took an afternoon because no test pointed at that page. The general suite eventually went red, but with failures scattered across unrelated scenarios that all happened to pass through the broken page. You want a sentinel: one scenario that watches this exact page for this exact failure and names it the moment it returns.

### Solution

```gherkin
  Scenario: Imports page loads without chunk errors
    Given I am on "/admin/imports"
    Then there should be no JavaScript errors
    And JavaScript errors should not match "chunk load error"
```

The first assertion is the general gate -- the error listener has been recording since the page opened, so a chunk that failed during initial load is already in the list by the time navigation settles. The second line looks redundant (if the first passes, the list is clean), but it earns its keep over time. Chunk failures have a habit of landing on ignore lists during incidents -- "mute it until the CDN fix ships" -- and a suite-wide `WEBSHIP_JS_ERROR_IGNORE` entry would silence the general check here too. The pattern step reads the unfiltered list, so the sentinel keeps biting even after someone mutes the noise globally. And when it fails, the message says `JavaScript errors matched /chunk load error/` -- triage starts with the diagnosis already written.

**At scale:** every project in a portfolio has one or two pages like this. Give the sentinel scenarios a shared tag (`@sentinel`) and run them first in CI -- they are cheap, they fail fast, and a red `@sentinel` run tells the reviewer "known failure mode returned" before the full suite finishes. The tag also gives delivery managers a clean answer to "which known risks do we actively watch?": grep the portfolio for `@sentinel` and read the scenario names.

---

## Recipe 442: Catching Synchronous Validation Errors

### Problem
Your signup form rejects bad input with a clean red message -- and also throws a synchronous error from the validation library on every rejection. Nothing visible breaks, so it shipped, and the team has been stepping around it for months. Then a real crash lands on the same page, and the engineer debugging it has to fish one genuine `TypeError` out of a console flooded with expected validation throws. Noise you tolerate today is camouflage for the bug you hunt tomorrow.

### Solution

```gherkin
  Scenario: Form rejects bad input without leaking errors
    Given I am on "/signup"
    When I fill in "email" with "not-an-email"
    And I press "Sign up"
    Then ".error-message" should be visible within 1 second
    And there should be no JavaScript errors
```

The timing works out because two waits stack in your favour. The visibility check is web-first -- it polls every 100ms for up to the 1 second you asked for. And the press itself is a state-changing step, so webship-js's AfterStep hook has already run `smartSettle(page, 1500)` before the next step starts: ajax counter at zero, pending timers at zero, DOM quiet. By the time the final assertion reads the list, the whole validation cycle -- synchronous throw included -- has fired and been captured. A synchronous throw from the library arrives as a `pageerror` entry; a `console.error` it logs arrives at level `error`. Both fail the last step, which is exactly the contract: "show the user a friendly message AND stay silent in the console." Both halves matter.

---

## Recipe 443: Differentiating Errors and Warnings

### Problem
Your design system pins a framework version that prints deprecation warnings on every page, and the upstream fix is two quarters away. Meanwhile your own `TypeError`s must never ship. If the console policy cannot tell those two apart, it ends up in one of two bad places: a check that nags on every run until the team stops reading it, or no check at all. The Tuesday before release is exactly when someone proposes the second option.

### Solution

```gherkin
  Scenario: Errors are forbidden, warnings ignored
    Given I am on "/"
    Then there should be no JavaScript errors
    # No warning check -- we accept deprecation warnings from third-party libs
```

The scoping does its work at two layers. The explicit step only evaluates error entries, so warnings cannot fail it. And with the default `WEBSHIP_JS_ERROR_LEVELS=error`, warning-level console messages are never even captured -- they cost no memory, no report space, and cannot leak into a failure message. The comment line is doing real work too: it tells the next reader the missing warning check is a decision, not a gap. When the upstream fix finally lands, tighten in two moves -- add `there should be no JavaScript warnings` and widen capture with `WEBSHIP_JS_ERROR_LEVELS=error,warning` on the runs where you want it enforced.

---

## Recipe 444: Watching for Specific Error Types

### Problem
Your checkout throws a custom `BillingError` when the gateway declines a card -- the error boundary catches it and shows the banner, exactly as designed. But the decline scenario *triggers* that error on purpose, so `there should be no JavaScript errors` can never pass here. The first person to hit this usually deletes the console check from the scenario entirely -- and now the one page where money changes hands is the one page with no console guard. You want surgical tolerance: the deliberate `BillingError` is fine, anything else still fails.

### Solution

```gherkin
  Scenario: Decline scenario emits exactly one BillingError
    Given the URL "**/api/payment" returns status 402 with body "Payment Required"
    When I am on "/checkout"
    And I click "Place Order"
    Then ".error-banner" should be visible within 3 seconds
    And JavaScript errors should not match "^(?!.*BillingError).*"
```

Three details make this work. The mock comes first and uses a `page.route()` glob -- `**/api/payment` matches the call wherever the page is served from, while a bare `/api/payment` would be compared against the full URL and never intercept anything. The 402 with a body gives the frontend a real decline to chew on. And the regex is the surgical part: `^(?!.*BillingError).*` anchors a negative lookahead at the start of each captured message, so the pattern matches any error that does **not** mention `BillingError` anywhere -- and the step fails on a match. The deliberate error sails through; an accidental `TypeError` trips the wire. The step tests each captured message independently, so if the decline path throws `BillingError` *and* a stray `TypeError`, the `TypeError` still fails the scenario -- which is precisely what you want.

---

## Recipe 445: Asserting Warning Count

### Problem
The dashboard carries three known deprecation warnings -- documented, ticketed, waiting on a vendor release. The team's instinct is to ignore warnings wholesale until that lands, but "ignore warnings" is how three quietly becomes seventeen and nobody can say which fourteen are new. You want a budget: the known three pass, the fourth one fails the build and has to be explained in review.

### Solution
Write a custom step over the world's captured list:

```js
const { Then } = require('@cucumber/cucumber');
const { friendly } = require('./webship');

Then(/^there should be at most (\d+) JavaScript warnings?$/, function(max) {
  const warnings = (this._jsErrors || []).filter((e) => e.type === 'console.warning');
  if (warnings.length > parseInt(max, 10)) {
    throw friendly(`Expected at most ${max} warnings, got ${warnings.length}:\n${warnings.map(w => w.message).join('\n')}`);
  }
});
```

Then in the feature:

```gherkin
  Scenario: Dashboard warning budget is three
    Given I am on "/dashboard"
    Then there should be at most 3 JavaScript warnings
```

The world keeps one chronological list, `this._jsErrors`, with entries shaped `{ type, message }` -- `pageerror` for uncaught exceptions, `console.warning` for warnings, and so on. The built-in `there should be no JavaScript warnings` step does exactly this filter with a threshold of zero; the custom step just makes the threshold a parameter. Two prerequisites: warnings must actually be captured (`WEBSHIP_JS_ERROR_LEVELS=error,warning` -- the default `error` records none), and the step file must live in `tests/step-definitions/` so `require('./webship')` resolves and cucumber-js loads it.

**At scale:** budgets beat bans when you inherit projects in different states of repair. Onboard each project with its budget set to its current count -- day-one green, no heroics required -- then ratchet the number down as tickets close. A budget that never tightens is decoration; review the numbers quarterly the way you review dependency versions, and let the feature file diff (`at most 3` becoming `at most 1`) document the cleanup for free.

---

## Recipe 446: Tracing Errors With Source Location

### Problem
The captured list says `TypeError: t is not a function`. Minified, anonymous, owned by nobody -- you cannot even file it against the right team yet. Before this becomes a debugging session, you want every scrap of context the capture pipeline holds: what kind of error it was, what it said in full, and what fired around it.

### Solution

```gherkin
  Scenario: Capture error source
    Given I am on "/"
    Then print JavaScript errors
```

Each printed entry is numbered in firing order and prefixed with its capture type: `[pageerror]` for uncaught exceptions, `[console.error]` for explicit logging, `[console.warning]` if you widened the levels. The prefix alone routes triage -- an uncaught exception is a crash, while a `console.error` is a developer's own message that survived to production; those go to different owners. The message text is everything the browser provided, untruncated, and the print step deliberately bypasses `WEBSHIP_JS_ERROR_IGNORE`, so you see muted noise too.

When the message itself is minified garbage, the fix lives in the app, not the test: serve source maps or an unminified build to the environment under test and the same step starts printing readable function names. To pin down *which step* introduced the error, pair this with the AfterStep counter from Recipe 448 -- the print tells you what fired, the counter tells you when.

---

## Recipe 447: Asserting Only the First Page Error

### Problem
One broken module fires three errors in a chain: the real failure, then two from code that expected the module to exist. A regression guard that matches against *all* captured errors is brittle here -- refactor the page and the cascade reshuffles, and the test starts failing on a symptom instead of the cause. You want the assertion pinned to the error that starts the chain.

### Solution
```gherkin
  Scenario: First error matches expected pattern
    Given I am on "/broken"
    Then print JavaScript errors
```

In a custom assertion:
```js
Then('the first JavaScript error should match {string}', function(pattern) {
  const errors = this._jsErrors || [];
  if (!errors.length) throw friendly('No JavaScript errors recorded.');
  const re = new RegExp(pattern);
  if (!re.test(errors[0].message)) {
    throw friendly(`First error did not match /${pattern}/. Got: ${errors[0].message}`);
  }
});
```

The listeners push entries the instant events fire, so `this._jsErrors[0]` is chronologically first across both channels -- uncaught exceptions and console output interleave in true order. Keep the `print` step in the scenario while the page is still under investigation; swap it for the custom assertion once you know which root-cause pattern to pin.

One subtlety worth knowing: the built-in assertion steps set a flag (`this._jsErrorsAsserted = true`) that tells the automatic end-of-scenario check "already handled, stand down." A custom step does not set it -- so under `WEBSHIP_JS_ERROR_MODE=fail`, a scenario that deliberately triggers an error would pass your assertion and then fail the automatic check anyway. If the recorded errors are expected, set the flag inside your step.

---

## Recipe 448: Hook-Level Console Recording

### Problem
The print step tells you *what* fired; it doesn't tell you *when*. A twelve-step scenario with four captured errors leaves you bisecting by hand -- commenting out steps, re-running, narrowing down. What you actually want is a log line that says "the count jumped right after the coupon click" so the guilty step names itself.

### Solution
Use a step hook:

```js
const { AfterStep } = require('@cucumber/cucumber');

AfterStep(function(step) {
  const errors = this._jsErrors || [];
  if (errors.length) {
    console.log(`After step "${step.pickleStep.text}", errors recorded: ${errors.length}`);
  }
});
```

This pinpoints which step introduced the first error -- invaluable for cascading failures.

The timing is more reliable than it looks. Webship-js's own AfterStep hook runs `smartSettle(page, 1500)` after every state-changing step, waiting for the ajax counter, pending timers, and DOM mutations to go quiet -- so the asynchronous fallout of a click (fetch callbacks, `setTimeout` chains) has usually landed in the list before your hook reads it. Errors get attributed to the step that caused them, not the step that happened to be running when a stray microtask flushed. Want it sharper? Stash the previous length on the world and log only the delta -- a line that reads `+3 errors after "I click 'Apply Coupon'"` ends the bisection before it starts.

---

## Recipe 449: Resetting Errors Mid-Scenario

### Problem
A resilience scenario does something rude on purpose: it kills the payment API, watches the error banner appear, then retries and expects a clean recovery. The deliberate failure necessarily logs console errors -- so a scenario-end console check can never pass, and the usual workaround (drop the check) throws away exactly the guarantee you care about: that the *recovery* path is silent. You need a line in the sand, not a blindfold.

### Solution

```js
When('I clear the recorded JavaScript errors', function() {
  this._jsErrors = [];
});
```

In the feature:

```gherkin
  Scenario: Recovery flow is clean after deliberate failure
    Given I am on "/checkout"
    When the URL "**/api/payment" returns status 500 with body "Internal Server Error"
    And I click "Place Order"
    Then ".error-banner" should be visible within 3 seconds
    When I clear the recorded JavaScript errors
    And I click "Try again"
    Then ".success-banner" should be visible within 5 seconds
    And there should be no JavaScript errors
```

Errors and warnings share the one `this._jsErrors` list, so a single reset clears both. The clear step splits the scenario into two acts: everything before it may scream, everything after it must be silent. And because the automatic end-of-scenario check reads the same list, the reset also satisfies a suite running under `WEBSHIP_JS_ERROR_MODE=fail` -- the policy only ever sees the post-reset entries. Order matters: clear *after* asserting the failure state (the banner check), never before, so the deliberate error is still on record while you verify the page handled it. The mock uses the `**/api/payment` glob so `page.route()` intercepts the call regardless of host, and the 500 carries a body so the frontend's error handler has something real to parse.

---

## Recipe 450: The Standard Health Block

### Pattern
Most teams settle on a four-line health block at the end of every scenario:

```gherkin
  Then there should be no JavaScript errors
  And there should be no broken images
  And the page should pass an accessibility audit at level "AA"
  And the page should have no critical accessibility violations
```

Together they cover: code correctness (no errors), asset correctness (no broken images), accessibility (level AA), and absence-of-blockers (no critical a11y violations).

Add the block to a `Background:` if you want it on every scenario in a file. Add it to a global `After` hook if you want it everywhere. The cost is small. The value is enormous on the day a deploy breaks something subtle.

---

The JavaScript-errors family and the friendly-error pipeline are the two features tester-friendliness depends on most: one says **what is wrong in the page**, the other says **what is wrong with the test in plain English**. Both reduce the gap between "my test failed" and "I know what to fix."

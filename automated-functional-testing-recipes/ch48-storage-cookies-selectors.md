# Chapter 48: Storage, Cookies, and Selectors -- Deep Dive

---

Three subsystems that sit underneath the rest of the test suite -- the **cookie jar**, the **storage trio** (localStorage, sessionStorage, IndexedDB cleanup), and the **named-selector registry**. Webship-js 2.0 expanded all three into rich families of steps. This is the deep-dive chapter that covers every one, with practical recipes.

Here is why these three deserve a chapter of their own: when a suite that has been green for months starts failing in ways nobody can explain, the root cause is almost never the assertion that failed. It is a cookie that leaked from a previous scenario, a localStorage key the app wrote and nobody cleared, or a selector that three feature files each define slightly differently. If you've worked through Chapter 10 (authentication) and Chapter 34 (advanced selectors), this chapter goes one layer deeper: variant cookie matchers, storage cleanup as a hook pattern, named selectors loaded from JSON, runtime selector additions, and the layout assertions that build on all of it.

---

## Cookie Family Overview

<!-- diagram: webship-architecture -->

Cookies have **four orthogonal questions** you might ask about them:

1. Is the name exact, or substring? (`name "session"` vs `name containing "sess"`)
2. Is the value exact, or substring? (`value "abc123"` vs `value containing "abc"`)
3. Should it exist, or not? (`should exist` vs `should not exist`)
4. Are we asserting, or setting?

Webship-js ships dedicated steps for every combination. Four name-and-value combinations, each with a `should exist` and a `should not exist` form, gives eight. Add the name-only checks (exact and containing, positive and negated) and you land on **twelve cookie matchers**, plus three setters (`is set to`, `is removed`, `all cookies are cleared`) -- enough to cover any cookie-related test you'd write.

All twelve matchers read the jar the same way: through Playwright's `context.cookies()` API, at the browser-context level. That one design decision matters more than it looks. `context.cookies()` sees **every** cookie in the context -- including `HttpOnly` cookies that page JavaScript and `document.cookie` can never read. Since real session cookies are almost always `HttpOnly`, a cookie assertion that ran through `document.cookie` would be blind to exactly the cookies you care about most. These steps are not.

---

## Recipe 566: Setting a Cookie

### Problem
Your site reads `theme=dark` from a cookie and renders the dark layout server-side. The only way to create that cookie through the UI is a four-click trip through the preferences page -- which means every dark-mode test spends its first ten seconds testing the preferences page instead of the thing it's actually about. You want the cookie to simply exist before the page loads.

### Solution

```gherkin
Feature: Theme cookie

  Scenario: Dark theme renders when cookie is set
    Given the cookie "theme" is set to "dark"
    When I am on "/"
    Then "body" should have class "theme-dark" within 1 second
```

The cookie is set before navigation. The page sees it on first render.

Under the hood the step calls Playwright's `context.addCookies()`. It needs a domain to attach the cookie to, so it looks at the current page URL -- and if the page is still on `about:blank` (which it is, at the start of a scenario), it falls back to the hostname of your `launchUrl`. Path defaults to `/`. Because the cookie lives at the browser-context level, not the page level, it is already in the jar when the first request goes out -- the server receives it in the `Cookie` header of the very first HTML request, which is why server-rendered dark mode works on first paint. The closing assertion is a web-first matcher: it polls the page every 100ms for up to one second, so a class applied by a late-running script still passes.

**Pro Tip:** This pre-navigation trick works for cookies because they are context-scoped. It does *not* work for localStorage, which belongs to an origin the browser hasn't visited yet -- see Recipe 574 for the storage equivalent.

---

## Recipe 567: Removing a Cookie

### Problem
A user's session expires while they have a tab open. They click something, and instead of a polite redirect to the login page they get a white screen and a stack trace -- you know, because that exact ticket landed on your desk last quarter. To test the fix you need a logged-in browser whose session cookie suddenly vanishes, mid-scenario, while everything else stays intact.

### Solution

```gherkin
  Scenario: Expired session redirects to login
    Given I restore the auth state from "tests/auth/webmaster.json"
    And I am on "/admin/content"
    When the cookie "session" is removed
    And I reload the page
    Then the path should be "/user/login"
```

The cookie is removed without clearing the rest of the cookie jar -- preserves test data set in earlier steps.

Two pieces of machinery are doing work here. The `restore the auth state` step (from `auth.steps.js`) opens a fresh browser context seeded from a saved `storageState` JSON snapshot -- cookies plus localStorage from a real login that happened once, earlier -- and re-installs the webship-js init script so smart waits keep working. That's how the scenario is "logged in" without walking through the login form. The removal step then reads the full jar via `context.cookies()`, filters out the one cookie by name, clears everything, and re-adds the survivors. The result is surgical: language preference, consent flag, A/B bucket all stay; only the session dies. The reload sends the next request without the session cookie, and the server does whatever your fix says it should do.

**At scale:** the snapshot files are the part that pays off across projects. Keep one per role per project -- `tests/auth/admin.json`, `tests/auth/editor.json` -- generated by a one-shot `@auth-setup` scenario you re-run after every credential rotation. Twenty projects with the same layout means anyone on the team can open any suite and know exactly where authentication comes from, and your CI secrets live in one setup scenario instead of being sprinkled through every feature file.

---

## Recipe 568: Clearing All Cookies

### Problem
The cookie-banner test passes when you run the feature file alone and fails when the whole suite runs -- because some earlier scenario accepted the banner, and your "first visit" is no longer a first visit. It's the classic Tuesday-before-release discovery: the test was never wrong, the starting state was.

### Solution

```gherkin
  Scenario: Anonymous visit
    Given all cookies are cleared
    When I am on "/"
    Then ".cookie-banner" should be visible within 2 seconds
```

The clear is total -- session, preferences, consent flags, everything. Use sparingly; most tests benefit from per-scenario isolation via the Cucumber lifecycle, not manual clears.

The step is one line of Playwright: `context.clearCookies()`. Worth knowing before you reach for it: webship-js already opens a **fresh browser context for every scenario** in its `Before` hook, and a fresh context has an empty jar. So in a standard suite the example above is belt-and-braces -- the real use cases are mid-scenario resets (log in, do something, then verify the anonymous view of the same page) and scenarios that restored an auth snapshot and now need to shed it. There is also a heavier sibling: `Given I am an anonymous user` clears cookies *and* navigates to your `launchUrl` in one step, which reads better when "fresh visitor on the homepage" is the actual precondition.

**At scale:** if a scenario only passes after a manual cookie clear, treat that as a smell, not a fix. Across 20 projects, suites that depend on scenario *order* are the ones that go green locally and red in CI when parallel workers shuffle execution. The per-scenario fresh context is your isolation guarantee -- write every scenario as if it runs first, and reserve explicit clears for mid-scenario state changes you can point to in the feature text.

---

## Recipe 569: Asserting Cookie Exists by Exact Name

### Problem
A login that *looks* successful -- right redirect, right welcome message -- but never set the session cookie will bite you three pages later, when the user is mysteriously anonymous again. The new hire who debugs this opens DevTools, types `document.cookie`, sees nothing, and loses an afternoon -- because session cookies are `HttpOnly` and invisible to page JavaScript. You want the test to check the jar directly, at the level where the cookie actually lives.

### Solution

```gherkin
  Scenario: Login sets the session cookie
    Given I am on "/user/login"
    When I fill in "username" with "normal_user"
    And I fill in "password" with "dD.123123ddd"
    And I press "Log in"
    Then a cookie with the name "SESS_a1b2c3" should exist
```

`should exist` is the strict, name-based existence check.

The step fetches the live jar with `context.cookies()` and does an exact `name ===` comparison. Because it reads at the browser-context level, it sees `HttpOnly` and `Secure` cookies that `document.cookie` cannot -- which is precisely what makes it useful for session cookies. On failure you get `Cookie "SESS_a1b2c3" was not found.` rather than a downstream symptom three steps later. One thing this step deliberately does *not* do is wait: it reads the jar at the moment it runs. That's fine here because `I press "Log in"` is a state-changing step, and webship-js runs `smartSettle` after it -- the response (and its `Set-Cookie` header) has landed before the assertion fires.

**Pro Tip:** the hash suffix in `SESS_a1b2c3` is environment-specific -- this exact-name assertion will pass on dev and fail on staging. If your suite runs against more than one environment, you want the next recipe.

---

## Recipe 570: Asserting Cookie by Partial Name

### Problem
Drupal's session cookie name includes a hash that changes per environment -- `SESS_a1b2c3` on dev, `SESS_x9y8z7` on staging. The login test you wrote against dev goes red the first time the pipeline points it at staging, and someone "fixes" it by copying the feature file and hard-coding the other hash. Now you maintain two tests that say the same thing. You want one test that works everywhere.

### Solution

```gherkin
  Scenario: Login sets a session cookie
    Given I am on "/user/login"
    When I fill in "username" with "normal_user"
    And I fill in "password" with "dD.123123ddd"
    And I press "Log in"
    Then a cookie with a name containing "SESS" should exist
```

`a name containing` does substring matching. Useful for cookies whose names are environment-dependent or library-generated.

Implementation-wise it's the same `context.cookies()` read as the exact matcher, but the comparison becomes `name.indexOf(partial) !== -1` -- a plain substring check, scanned across every cookie in the jar until one matches. That covers Drupal's per-site hash, framework-generated names (`csrftoken_`, `XSRF-`), and analytics cookies with versioned suffixes. The trade-off is honesty about looseness: `containing "SESS"` would also match a hypothetical `SESSION_DEBUG` cookie. Pick the longest stable fragment you have.

**At scale:** environment-portable assertions are what let one feature file serve your whole pipeline. The pattern that works across many projects: the suite takes its target from the `LAUNCH_URL` env var, and every assertion inside the features is written to pass on *any* environment -- partial cookie names, path assertions instead of full URLs, label-based field lookups. The moment a feature file contains an environment-specific literal, you've forked your suite without noticing.

---

## Recipe 571: Asserting Cookie Value

### Problem
The consent banner sets `consent=accepted` when the user clicks "Accept all" -- and the legal team has made it very clear that "the cookie exists" is not the same claim as "the cookie says accepted". A refactor that accidentally writes `consent=1` instead would pass an existence check and still put you out of compliance. You need the value pinned, exactly.

### Solution

```gherkin
  Scenario: Consent cookie value
    Given I am on "/"
    When I click "Accept all"
    Then a cookie with the name "consent" and the value "accepted" should exist
```

Both name and value match exactly.

The click step finds a link or button whose visible text is exactly "Accept all" -- under the hood it filters `a, button, [role="button"], input[type="submit"]` by text, so it works whether the banner ships a `<button>` or a styled link. The assertion then finds the cookie by exact name and compares the value with strict equality. When the value is wrong you get the actual vs. expected in the failure message -- `Cookie "consent" has value "1", expected "accepted".` -- which turns a compliance bug from an investigation into a one-line diff. This is the strictest of the twelve matchers; use it whenever the value is a contract, not an implementation detail.

---

## Recipe 572: Asserting Cookie Value by Substring

### Problem
A tracking cookie stores a JSON blob: `{"opt_in":true,"id":"abc"}`. The `id` changes every session, so an exact-value assertion fails on every run -- and the first instinct, asserting the whole JSON with a wildcard id, couples your test to field order and every future key the analytics team adds. You care about one fact: `opt_in` is true. Assert that slice and nothing else.

### Solution

```gherkin
  Scenario: Tracking opt-in is set
    Given I am on "/"
    When I click "Allow tracking"
    Then a cookie with the name "tracking" and a value containing "opt_in\":true" should exist
```

Substring matching on the value -- great for JSON payloads where the test only cares about a slice.

The matcher finds the cookie by exact name, then runs `value.indexOf(partial) !== -1` against the raw stored value. Note the `\"` in the Gherkin -- Cucumber's `{string}` parameter supports backslash-escaped quotes, so the actual substring searched for is `opt_in":true`, exactly as it sits inside the JSON. The recipe survives new keys, reordered fields, and rotating ids, because none of those touch the five characters you pinned.

**Pro Tip:** `context.cookies()` returns the value exactly as stored -- and many frameworks URL-encode cookie values, so your JSON may actually live in the jar as `%7B%22opt_in%22%3Atrue...%7D`. If the substring assertion fails against a cookie you can see in DevTools, encoding is the first suspect. Check the raw value once (a temporary custom step that prints `await this.context.cookies()` does the job), then match a fragment that survives the encoding your app uses.

---

## Recipe 573: The Four-Way Cookie Matrix

Teams that test cookies across several projects keep reinventing this table -- usually badly, in a Slack thread, the day someone needs the negated-substring form and can't remember whether it exists. Here is the full matrix in one place:

| Name | Value | Step |
|------|-------|------|
| exact | exact | `a cookie with the name "X" and the value "Y" should exist` |
| exact | contains | `a cookie with the name "X" and a value containing "Y" should exist` |
| contains | exact | `a cookie with a name containing "X" and the value "Y" should exist` |
| contains | contains | `a cookie with a name containing "X" and a value containing "Y" should exist` |

All four have `should not exist` negations -- that's eight. Add the name-only pair from Recipes 569-570 (`a cookie with the name "X" should exist` / `a name containing "X" should exist`) and their negations, and you reach the full twelve. Every one of them is the same two-line pattern inside: read `context.cookies()`, then match with `===` or `indexOf` depending on the row you picked.

Use the most specific form your test can stand. Looser matches hide more bugs: `a name containing "consent"` happily passes while the real cookie is `consent_v2_legacy_do_not_use`. The negated forms deserve special respect -- `a cookie with the name "tracking" should not exist` *after declining consent* is one of the few assertions that catches a privacy regression before your users (or a regulator) do.

**At scale:** put this table in your team's testing conventions doc and reference it in review. When 10 suites across 10 projects each assert cookies in their own dialect -- some through custom steps, some through `document.cookie` evaluations -- review can't tell a deliberate loose match from a lazy one. A shared rule like "exact name and value for contract cookies, substring only for generated names, negation required for consent declines" makes every cookie assertion reviewable by anyone on the team, not just the person who wrote it.

---

## Recipe 574: LocalStorage Setting and Removal

### Problem
Your site shows a "Continue reading" link pointing at whatever the visitor last read -- the app keeps a `last_page` key in localStorage. Testing it through the UI means three navigations: open the article, leave, come back. And the first person who copies the cookie recipe from earlier in this chapter -- seed the value, *then* navigate -- gets a failure that says `Access is denied for this document` and loses an hour to it. Storage is not cookies. It needs a page first.

### Solution

```gherkin
  Scenario: Last-viewed page is restored
    Given I am on "/"
    And the local storage "last_page" is set to "/articles/42"
    When I reload the page
    Then the link "Continue reading" with the href "/articles/42" should exist
```

Removal is the same shape -- one key gone, every other key untouched:

```gherkin
  Given the local storage "last_page" is removed
```

The set step is a single `page.evaluate()` call running `localStorage.setItem(key, value)` on whatever page is currently open. That one fact explains the step order, and it is the exact mirror of Recipe 566's Pro Tip. Cookies live at the browser-context level, so you can plant them before any navigation. localStorage belongs to an *origin* -- and at the start of a scenario the page is still parked on `about:blank`, which has no origin storage at all; touching `localStorage` there throws. So the working sequence is always: navigate (now the origin exists), seed the key, reload (the app boots again and reads what you planted). The reload is not optional -- the app finished its startup before you wrote the key, so it has to start over to notice. One more thing the step will not do for you: stored values are always strings. JSON-encode objects yourself -- Recipe 594 shows that in anger.

---

## Recipe 575: Clearing All LocalStorage

### Problem
You are testing the first-visit welcome tour, but by the time your scenario reaches the interesting part, the app has already written `tour_seen=true` -- your own earlier steps triggered it. You need to wipe storage *mid-scenario* and watch the page behave like it has never met this visitor. The instinct is to clear before navigating, the way the cookie recipes do -- and that is exactly the order that fails.

### Solution

```gherkin
  Scenario: Fresh visit, empty storage
    Given I am on "/"
    And local storage is cleared
    When I reload the page
```

Equivalent for session storage:

```gherkin
  Given session storage is cleared
```

The step runs `localStorage.clear()` inside the page -- per-origin, same `about:blank` rule as Recipe 574, which is why the navigation comes first. Two scoping facts keep this honest. First, "all" means all *for the current origin*: `app.example.com` and `admin.example.com` hold separate storage, and clearing one says nothing about the other -- cookies, by contrast, can span subdomains. Second, you need this step less often than you think. Webship-js opens a fresh browser context for every scenario, and a fresh context starts with empty storage everywhere -- so the clear at the top of a scenario is usually redundant. Reach for it for mid-scenario resets like the tour example, the same advice Recipe 568 gives for cookies: if a scenario needs a clear at the top just to pass, something else is leaking, and the clear is hiding it.

---

## Recipe 576: Session Storage Set and Remove

### Problem
Your checkout wizard keeps its progress in sessionStorage under `wizard_step`, so a refresh keeps the user where they were but a fresh tab starts over. QA needs to land on step 3 without clicking through steps 1 and 2 every single run -- and last sprint someone wrote that walk-through as a fixture, then watched it break every time the step-2 form changed. Seed the key, open the page, and the wizard resumes where you told it to.

### Solution

```gherkin
  Scenario: Wizard resumes at step 3
    Given the session storage "wizard_step" is set to "3"
    When I am on "/wizard"
    Then ".wizard-step-3" should be visible within 2 seconds
```

Session storage is per-tab -- it clears on tab close. Use it for ephemeral state that shouldn't survive a restart.

The set step is a `page.evaluate()` running `sessionStorage.setItem(key, value)` -- the sessionStorage twin of Recipe 574's `localStorage.setItem`, and it lives under the same origin rules. The example reads as if it sets the key *before* navigating, but read the order again: the `Given` runs against whatever page is open, and at scenario start that is `about:blank`, which has no origin and throws on any storage access. In practice you navigate first, seed the key, then trigger the read -- the same navigate-seed-reload dance Recipe 574 spells out. The removal form is `Given the session storage "wizard_step" is removed`, and like its localStorage sibling it deletes one key and leaves the rest of the tab's storage alone. The values are strings; JSON-encode anything richer yourself.

**At scale:** sessionStorage and localStorage look interchangeable until you run them across a fleet, and then the difference bites. sessionStorage dies with the tab, so a scenario that seeds it survives a `reload` but not a `start/stop video recording` step -- that rebuilds the browser context and takes the tab (and its session storage) with it (see the video chapter). When you template wizard tests across 20 product suites, write a one-line comment in the feature noting which storage the app actually uses; the new hire who copies your scenario into a localStorage-backed wizard otherwise spends an afternoon wondering why a seeded value vanished on the next tab.

---

## Recipe 577: Storage Hook Pattern

### Problem
A suite that has been green for a month starts failing one scenario in three, and only in CI, never locally. The cause is always the same shape -- one scenario writes a localStorage key, the next scenario assumes a clean slate, and which one runs first depends on how the parallel workers shuffled the order that night. You want a single place that guarantees every scenario starts with nothing in cookies or storage, so order stops mattering.

### Solution

```js
const { Before } = require('@cucumber/cucumber');

Before(async function() {
  await this.page.context().clearCookies();
  await this.page.evaluate(() => {
    localStorage.clear();
    sessionStorage.clear();
  });
});
```

Drop into your global hooks file. Every scenario starts truly clean.

Here is the thing worth knowing before you add this: webship-js already opens a **fresh browser context for every scenario** in its own `Before` hook, and a fresh context has an empty cookie jar and empty storage everywhere. So `clearCookies()` here is belt-and-braces. The `localStorage.clear()` and `sessionStorage.clear()` calls run inside the page via `page.evaluate()`, and they are per-origin -- they wipe storage for the origin the page is currently sitting on, nothing more. At the very top of a scenario the page is still on `about:blank`, which has no origin storage and throws when you touch `localStorage`; this hook survives only because it runs after the framework's own setup has navigated somewhere with an origin. If you ever see it blow up with `Access is denied for this document`, that is the `about:blank` problem -- Recipe 591 wraps each clear in a `try/catch` precisely so a no-origin page degrades quietly instead of failing the run.

**At scale:** this hook is the single most copied snippet across a multi-project test estate, which is exactly why it should live in one shared hooks file you publish once, not re-typed into 20 repos that then drift. When every suite isolates state the same way, a red scenario in CI means a real regression, not a leaked cookie -- and that is the difference between a PMO dashboard people trust and one they have learned to ignore. The corollary is a review rule: a scenario that only passes because *another* scenario left state behind is a bug in the test, and this hook is what makes that bug impossible to write by accident.

---

## Recipe 578: Defining a Single Named Selector Inline

### Problem
The login button is buried behind `nav.primary a[data-role='auth-cta']:not(.disabled)`, and that exact string is now pasted into nine scenarios. The day the design team renames `data-role`, you are doing find-and-replace across nine files and praying you caught them all. You want one short name -- `login-button` -- that every step can use, defined once.

### Solution

```gherkin
  Scenario: Login uses named selector
    Given I add "login-button" selector for "nav.primary a[data-role='auth-cta']:not(.disabled)" css selector
    When I am on "/"
    And I click "login-button"
    Then ".user-menu" should be visible within 2 seconds
```

`add ... css selector` registers the name in the per-scenario registry. Subsequent steps can refer to `"login-button"` instead of the full CSS.

The registry is part of the world's `selectors` config -- a plain object shaped `{css: {}, xpath: {}}` that lives for the life of one scenario. This step writes `"login-button"` into the `css` map. From then on, every step that takes a selector tries the registry first: when `I click "login-button"` runs, the resolver finds the name, swaps in the full CSS, and hands that to Playwright's `getByRole`/locator machinery. A name that *isn't* in the registry falls straight through and is treated as a literal selector, which is why `.user-menu` on the next line just works without being registered. The registration is per-scenario by design -- the next scenario starts with whatever the world bootstraps (the files in `worldParameters.selectors.files`) and none of your inline additions, so one scenario can never pollute another's vocabulary.

**At scale:** inline `add ... selector` is the right tool for a selector that genuinely belongs to *one* scenario. The moment a name shows up in a second scenario, it has earned a home in a shared JSON file (Recipe 580) instead. The pattern that holds up across many projects is a layered vocabulary: a shared cross-project file for the handful of selectors every site has (`main-nav`, `search-box`), a per-project file for that product's quirks, and inline additions only for the genuinely one-off. Review can then ask one question -- "why is this selector inline instead of in the file?" -- and usually the answer reveals either a one-off (fine) or a missed chance to share (fix it).

---

## Recipe 579: Defining Multiple Selectors in Bulk

### Problem
The complex-form scenario needs ten named selectors, and ten separate `add ... selector` lines push the actual test -- the three steps that matter -- below the fold of anyone's editor. A reviewer scrolling past ten setup lines to find one assertion stops reading carefully, which is exactly when bugs slip through. You want the whole vocabulary declared in one readable block, then the test.

### Solution

```gherkin
  Scenario: Complex form uses many selectors
    Given I define css selectors:
      | name           | selector                                          |
      | login-button   | nav.primary a[data-role='auth-cta']:not(.disabled) |
      | name-field     | input[name='username']                            |
      | password-field | input[name='password']                            |
      | submit-button  | button[type='submit'].btn-primary                 |
    When I am on "/user/login"
    And I fill in "name-field" with "user"
    And I fill in "password-field" with "pass"
    And I click "submit-button"
```

One step, a data table, every selector registered. Use this when a scenario has five or more named selectors.

`I define css selectors:` takes a Cucumber data table -- two columns, `name` and `selector` -- and loops every row into the same per-scenario `css` registry that Recipe 578 writes to one entry at a time. There is no behavioural difference between this and four `add` steps; it is the same map ending up with the same keys. What changes is readability, and that is the whole point: the resolver still tries the registry first for every selector-taking step, so `name-field` resolves through `input[name='username']` to label-first field resolution, and `submit-button` reaches `getByRole` through `button[type='submit'].btn-primary`. The xpath twin is `I define xpath selectors:` (Recipe 582), writing to the separate `xpath` map.

**At scale:** a bulk-define block at the top of a scenario is a sign you are about to share these selectors. Treat the table as a staging area -- the day a second scenario needs the same four names, lift the table into a JSON file (Recipe 580) and let `worldParameters.selectors.files` auto-load it. Across 20 suites the rule of thumb is simple: inline single adds for one-offs, a define-table when a single scenario needs many that are still genuinely local, and a JSON file the instant anything is reused. The table format also diffs cleanly in code review, so a selector change reads as one row changed, not a buried edit inside a long Gherkin line.

---

## Recipe 580: Loading Selectors From a JSON File

### Problem
The same `login-button` selector is defined inline in a dozen scenarios across four feature files, and they have already drifted -- two of them still point at the old `data-role`, and nobody noticed until a flaky login test turned out to be two different selectors wearing the same name. You want the selector defined in exactly one place, read by every scenario automatically, so "the login button" means one thing in the whole suite.

### Solution
Create `tests/selectors/auth.json`:

```json
{
  "css": {
    "login-button": "nav.primary a[data-role='auth-cta']:not(.disabled)",
    "name-field":   "input[name='username']",
    "password-field": "input[name='password']"
  }
}
```

Reference it in `worldParameters.selectors.files`:

```js
// cucumber.js
module.exports = {
  default: {
    worldParameters: {
      selectors: { files: ['tests/selectors/auth.json'] }
    }
  }
};
```

Now every scenario inherits the file's selectors. Override per-scenario with inline `add ... selector` steps when needed.

This is the registry's intended home. Each file is the same `{"css": {}, "xpath": {}}` shape as the in-memory registry, which is no accident -- at world setup webship-js reads every path in `worldParameters.selectors.files`, resolves it against `selectors.filesPath` (default `'./tests/selectors/'`), and merges the file's `css` and `xpath` maps into the scenario's starting registry. So by the time your first step runs, `login-button` is already a known name in every scenario, with zero Gherkin spent registering it. Inline `add ... selector` steps still work and layer on top of the file-loaded names, which is what "override per-scenario" means: a later `add` for an existing name wins for that scenario only, and the next scenario reverts to the file's version. Keep the file rooted under `tests/selectors/` and `files: ['auth.json']` resolves just as well as the full path.

**At scale:** the JSON file is the artifact that makes a selector vocabulary shareable across an entire estate, and it is plain JSON on purpose -- a non-developer can read it, a PO can confirm "login-button" points where they expect, and DevOps can swap it per environment without touching a single feature file. The pattern that holds across 20 projects: a shared base file (published as an npm package, Recipe 592) for the selectors every site has, plus a thin per-project file for that product's quirks, both listed in `files:`. A selector rename then becomes a one-line edit in one file that the whole suite picks up on the next run -- the opposite of the drift you started with.

---

## Recipe 581: Adding Selectors From a File Mid-Scenario

### Problem
Your admin-area scenarios need 30 admin-only selectors, but the other 200 scenarios in the suite never touch the admin UI. Dumping all 30 into the global `files:` list means every front-end test loads selectors it will never use, and the names start colliding with public-facing ones -- a `toolbar` in admin is not the `toolbar` on the marketing page. You want those selectors loaded only for the scenarios that actually need them.

### Solution

```gherkin
  @admin
  Scenario: Admin uses admin selectors
    Given I add selectors from "tests/selectors/admin.json" file
    When I am on "/admin"
    And I click "admin-toolbar-cta"
```

The file is loaded into the scenario's registry. The next scenario starts without it -- registry is per-scenario.

This step does the same file read as the global `files:` loader (Recipe 580) -- parse the JSON, merge its `css` and `xpath` maps into the registry -- but at step time, into one scenario's registry only. Because the registry is rebuilt fresh for every scenario from `worldParameters.selectors.files`, anything you load mid-scenario evaporates when that scenario ends; the next one starts with the global baseline and nothing else. That is the scoping you want for an admin file: the 30 admin names exist exactly where the `@admin` tag says they should, and a `toolbar` selector in `admin.json` can shadow a global `toolbar` for that scenario without leaking into the front-end tests that ran before or after.

**At scale:** the global-versus-mid-scenario choice is really a namespace-collision strategy across many projects. Selectors that every scenario uses belong in the global `files:` list; selectors scoped to one section -- admin, a checkout flow, an embedded third-party widget -- belong in their own file loaded behind a matching tag. Pair the load step with the same tag your CI uses to slice the run (`@admin`), and DevOps gets a clean story: the admin selector file only loads when admin scenarios run, so a front-end-only nightly pass never parses it. The tag, the file, and the section it covers all carry the same name, which is the kind of consistency that lets someone open the twentieth project's suite and immediately know where the admin selectors come from.

---

## Recipe 582: Defining XPath Selectors

### Problem
You need to click the Edit link in the row whose cells contain "Article" -- and CSS has no way to say "the parent row that contains this text". The team's first attempt was an `:nth-child` index, which broke the moment the content list re-sorted. This is one of the handful of cases where XPath earns its keep, and you want to name the ugly expression so the scenario stays readable.

### Solution

```gherkin
  Scenario: XPath registers as a named selector
    Given I define xpath selectors:
      | name        | xpath                                              |
      | article-row | //tr[td[contains(.,'Article')]]                    |
      | edit-link   | //tr[td[contains(.,'Article')]]//a[text()='Edit']  |
    When I am on "/admin/content"
    And I click "edit-link"
```

XPath and CSS selectors live in separate registries but are searched together at resolve time. The CSS registry wins on name collision -- prefer CSS for new selectors when both would work.

`I define xpath selectors:` is the structural twin of Recipe 579's CSS define-table, except every row lands in the world's `xpath` map instead of `css`. The two maps are deliberately separate, but the resolver checks both when a step hands it a name -- so `edit-link` resolves the same way whether it was registered as CSS or XPath, and Playwright is told which engine to use based on which map held the name. When the *same* name exists in both, CSS wins, which is why the guidance is to reach for XPath only when CSS genuinely can't express the relationship -- ancestor/text predicates like the one above. JSON files carry XPath the same way: a `{"xpath": {...}}` block in any file from Recipe 580 auto-loads alongside its `css` sibling.

**At scale:** XPath is the selector style that ages worst, so the registry is also a containment strategy. Across many projects, structural XPath like `//tr[td[contains(.,'Article')]]` is the first thing to break when a CMS upgrade reshapes its markup -- and when that expression is pasted inline in 15 scenarios, you re-fix it 15 times. Named in one JSON file, it is one edit. Make it a convention that XPath always goes in the file behind a name, never inline in a feature, with a one-word comment on *why* CSS wasn't enough; the next person upgrading the platform then has a single, documented list of the brittle selectors to re-verify, instead of hunting them across the whole suite.

---

## Recipe 583: Printing the Active Selector Registry

### Problem
A step fails with `Could not find "foo button"`, and you are certain you registered it. The usual culprits are invisible: a trailing space in the name, a file that didn't load because the path was wrong, or a JSON file that silently merged into the wrong map. Rather than guess, you want to see exactly what names the resolver actually has right now.

### Solution

```gherkin
  Scenario: Debug the registry
    Given I define css selectors:
      | name        | selector |
      | foo button  | .foo-btn |
    Then print css selectors
    And print xpath selectors
```

Both `print` steps dump the active registry to stdout. Use them during scenario authoring; remove before committing.

These two steps print the world's live `css` and `xpath` maps at the moment they run -- the merged result of everything: the files from `worldParameters.selectors.files`, any mid-scenario file loads (Recipe 581), and every inline `add`/`define` up to that line. That is what makes them the right debugging tool. If `foo button` is missing from the printed CSS map, you registered it somewhere that didn't take -- wrong file, wrong map, a typo in the name. If it is present but the step still fails, the name resolved fine and your CSS itself doesn't match anything on the page, which sends you to the DOM, not the registry. Because the maps print as plain key/value, a trailing space or a case mismatch jumps straight out at you.

**Pro Tip:** these are authoring-time steps, not assertions -- they print and pass, they never fail a build. Pull them out before you commit, the same way you would a `console.log`. A `print css selectors` left in a feature file clutters every CI run's stdout and trains your team to ignore output that should mean something. If you find yourself needing them often in one project, that is usually a sign the selector file isn't loading the way you think -- fix the `files:` path once rather than debugging it scenario by scenario.

---

## Recipe 584: CSS Property Assertions

### Problem
A theme refactor moved the brand colour into a CSS custom property, and three weeks later the primary call-to-action button quietly rendered grey on production because a variable name had a typo nobody caught. Visual review missed it; it was a subtle shade off. You want a test that pins the actual rendered colour of the button, so "the CTA is the brand blue" is a fact the suite checks, not a thing humans eyeball.

### Solution

```gherkin
  Scenario: Hover changes button background
    Given I am on "/"
    When I hover over "#cta-button"
    Then "#cta-button" should contain cssproperty "background-color" with value "rgb(53, 143, 200)"
```

CSS properties are read via `getComputedStyle()`, so the value is the resolved final colour, not the raw declaration. Match against the resolved form (`rgb(...)` not `#358FC8`).

The assertion reads the property off the element's *computed* style, which is the value the browser actually paints after cascade, custom properties, and media queries have all resolved. That is why you match `rgb(53, 143, 200)` and not `#358FC8` or `var(--brand)` -- by the time `getComputedStyle()` sees it, every layer has collapsed to a concrete `rgb()` (or `rgba()` if there is any alpha). It also means this catches the broken-variable bug from the Problem directly: a mistyped `var(--brnd)` falls back to its default and the computed colour changes, so the assertion fails with the actual vs. expected colour. The `hover over` step moves the real pointer to the element first, so a `:hover`-only colour is genuinely in effect when the property is read.

**At scale:** colour and computed-style assertions are powerful and brittle in equal measure, so spend them where a value is a contract, not everywhere. Across many projects, hard-coded `rgb(...)` literals scattered through features are the assertions that go red en masse the day a design system re-tokenizes its palette -- and then the failure is the test, not the product. The pattern that survives: assert computed style only on the handful of elements where the colour is the point (brand CTA, error state, the disabled affordance from the next recipe), and assert the rest through `should have class` so a token change flows through without touching a single feature. When you do pin a colour, name the expected value once and reference it, so a palette bump is one edit across the estate, not a sweep.

---

## Recipe 585: Negated CSS Property Assertions

### Problem
A form's Save button is supposed to look obviously dead until the form is valid, but a CSS regression left it rendering in full brand blue while still disabled -- so users kept clicking a button that did nothing, and the support queue filled up. The trouble with asserting the *grey* disabled colour directly is that the grey is itself a design choice that changes; what you actually care about is that the disabled button does **not** wear the active colour.

### Solution

```gherkin
  Scenario: Disabled button has different background
    Given I am on "/profile"
    Then "#save-disabled" should not contain cssproperty "background-color" with value "rgb(53, 143, 200)"
```

Useful for testing CSS state machines (`:hover`, `:focus`, `:disabled`) where the assertion is "this is not the default appearance".

This is the negation of Recipe 584: same `getComputedStyle()` read, but it passes when the computed value is *anything except* the one you named. That framing is the whole value. You are not pinning the disabled button to a specific grey -- which would couple your test to a colour the designers will tweak -- you are asserting it is visibly different from the active state, which is the user-facing contract. The same computed-style resolution applies, so the comparison is against the painted `rgb()` after the disabled styles have cascaded in. Negated property checks shine on CSS state machines generally: a focused input should *not* have the resting border, a `:hover` row should *not* match the default background.

**Pro Tip:** a negated check is looser by nature -- `should not contain cssproperty "background-color" with value "rgb(53, 143, 200)"` would pass even if the disabled button rendered hot pink. That is the right trade for asserting *difference* between states, but it means you should reach for the positive assertion of Recipe 584 for the one or two colours that are genuine brand contracts, and lean on negation everywhere the claim is really "not the default appearance".

---

## Recipe 586: Spatial Relations Between Elements

### Problem
A CSS change to the notification container moved success toasts from the top of the screen to the bottom, behind the sticky footer, where nobody saw them -- so users kept re-submitting forms that had already succeeded. Every text assertion still passed; the toast existed, it said the right thing, it was simply in the wrong place. You want a test that asserts *where* an element renders, not just that it exists.

### Solution

```gherkin
  Scenario: Toast position
    Given I am on "/"
    When I click "Save"
    Then the element ".toast" should be at the top of the viewport within 2 seconds
```

Other spatial steps include `centered in the viewport`, `displayed within a viewport`, `should appear after the text "X"`. Useful for layout tests where the assertion is "this is in the expected position" rather than "this text exists".

Position steps work off the element's bounding box -- Playwright's `boundingBox()` returns the element's `x`, `y`, `width`, and `height` relative to the viewport, and the step compares those numbers against the region you named. "At the top of the viewport" means the box sits within a band of the viewport's top edge; the registry's `offset` (default 60px, `WEBSHIP_SELECTORS_OFFSET`) is the tolerance that keeps a few pixels of slack from failing the check. The `within 2 seconds` makes it a web-first assertion: it re-reads the bounding box on a poll until the toast has actually animated into place, so a slide-in transition doesn't race the check. Because it reads geometry, it catches exactly the class of bug text assertions miss -- the element is present and correct but laid out wrong.

**At scale:** spatial assertions are sharpest when paired with the viewport sizing from Recipe 590, because "where does this render" has a different answer at every breakpoint. The pattern that pays off across many projects: a small set of layout smoke checks -- toast at top, primary CTA above the fold, sticky header pinned -- run as a `Scenario Outline` over your breakpoints, so a CSS change that only breaks the layout at `md` gets caught by the same handful of steps everywhere. Tune the `WEBSHIP_SELECTORS_OFFSET` tolerance once at the suite level rather than sprinkling pixel fudge factors through features; a single env-var value keeps every project's idea of "at the top" consistent and tweakable from CI without a code change.

---

## Recipe 587: Element Ordering

### Problem
A template change reversed the breadcrumb trail -- the current page rendered first and "Home" last -- and every link still worked, so it shipped. Users navigating by breadcrumb ended up one level too deep and complained the site was "confusing". Order is content too, and you want a test that asserts the trail reads Home, then Articles, then the title, in that sequence.

### Solution

```gherkin
  Scenario: Breadcrumb order
    Given I am on "/articles/42"
    # NOTE: the line below is intentionally wrong -- it claims Home comes
    # after Articles. Read the direction of "appear after" carefully:
    #   Then the text "Home" should appear after the text "Articles"
    # The correct assertions, top of trail first, are:
    Then the text "Articles" should appear after the text "Home"
    And the text "My Article" should appear after the text "Articles"
```

Ordering steps work on text node order in the DOM. They catch breadcrumb regressions, list-sort bugs, and table-row ordering issues.

The direction trips people up, so read it literally: `"Articles" should appear after the text "Home"` passes when "Articles" comes *later* in document order than "Home" -- so the top of the breadcrumb is the text you name *second*. Under the hood the step finds both text nodes and compares their position in the DOM using the document's natural traversal order (the same order `compareDocumentPosition` reports), so it reflects source order, not visual order. That distinction matters: CSS flexbox `order` or `row-reverse` can paint elements in a different sequence than they sit in the DOM, and this assertion follows the DOM. For a breadcrumb that is exactly right -- you want to know the markup is in the correct order so it reads correctly to a screen reader, regardless of styling.

**At scale:** ordering assertions are the cheapest guard against a whole family of regressions that text checks miss -- breadcrumbs reversed, a sorted list that quietly stopped sorting, table rows in the wrong sequence after a query change. Across many projects the high-value targets are the same: breadcrumbs, sorted result lists, step indicators, navigation menus. The catch to flag in review is the DOM-versus-visual gap -- if a project uses flexbox reordering for responsive layouts, an ordering assertion can pass while the rendered order is wrong, so reserve these for content where DOM order *is* the contract (accessibility, semantic sequence) and lean on spatial checks (Recipe 586) when the visual position is what matters.

---

## Recipe 588: Top-Offset Asserts

### Problem
The sticky header that holds the search box and cart icon stopped sticking after a `position` change in a shared layout component -- it scrolled away with the rest of the page, so on long articles the cart was unreachable without scrolling all the way back up. It looked perfect on a short page, which is how it passed review. You want a test that scrolls down and then checks the header is still pinned to the top.

### Solution

```gherkin
  Scenario: Header stays pinned
    Given I am on "/long-article"
    When I scroll to bottom
    Then the element ".sticky-header" should be displayed within a viewport with a top offset of 0 pixels
```

The top-offset variant asserts the element's vertical position relative to the viewport top. Use 0 for sticky headers, negative values for off-screen modals.

This builds on the bounding-box read from Recipe 586 but pins one number: the box's `y` relative to the viewport top, after the scroll. `I scroll to bottom` moves the page as far down as it goes, then the assertion re-reads the header's position -- and a genuinely sticky header reports `y` near 0 because it stays glued to the top edge while the document slides under it. A header that lost its stickiness scrolls off and reports a large negative `y` (or vanishes from the box entirely), so the check fails. The registry's `offset` (default 60px, `WEBSHIP_SELECTORS_OFFSET`) is the tolerance band, so "a top offset of 0 pixels" forgives the handful of pixels a real browser rounds to. Negative offsets let you assert the opposite -- an off-screen drawer parked just above the fold. Sticky regressions are a classic shared-component failure that a short-page smoke test never catches, so this one short scenario, run over your breakpoints (Recipe 590), pays for itself many times over -- it catches a bug that otherwise surfaces as scattered "I can't find the cart" tickets.

**Pro Tip:** keep the tolerance in the shared `WEBSHIP_SELECTORS_OFFSET` env var rather than fudging pixel values per feature. A single value means every project agrees on what "pinned to the top" means, and you can widen it once from CI if a particular browser image rounds positions differently, without editing a single feature file.

---

## Recipe 589: Setting Custom Breakpoints

### Problem
Your design system breaks at `md: 768` and `lg: 1024`, but webship-js defaults to `md 768 / lg 992` -- so a responsive test that "passed at lg" was actually testing 992px, a width your layout never targets, and missed a bug that only shows between 992 and 1024. You want your suite's breakpoint names to mean the same pixel widths your CSS does, defined once for the whole project.

### Solution
Configure breakpoints in `worldParameters`:

```js
// cucumber.js
module.exports = {
  default: {
    worldParameters: {
      breakpoints: {
        xs: 480,
        sm: 640,
        md: 768,
        lg: 1024,
        xl: 1280,
        xxl: 1600,
      },
    },
  },
};
```

Then use them in features:

```gherkin
  Scenario: Mobile layout
    Given I am viewing the site on a "mobile" device
    And I am on "/"
    Then ".desktop-nav" should be hidden
```

Or directly via pixel size:

```gherkin
  When I set the viewport to the "md" breakpoint
```

The named breakpoints make features readable; the pixel sizes make them precise.

`worldParameters.breakpoints` overrides the registry's built-in map -- the shipped defaults are `xs 375x667 / sm 576x800 / md 768x1024 / lg 992x768 / xl 1200x900 (default) / xxl 1400x900 / xxxl 1920x1080`, and your config replaces the widths so a name resolves to *your* design system's pixels. When a step says `set the viewport to the "md" breakpoint`, the resolver looks up `md` in this map and calls Playwright's `page.setViewportSize()` with the matching dimensions; from then on every layout, visibility, and spatial assertion is measured at that width. That is the whole point of naming them: `md` in a feature file reads as intent, and the one number behind it lives in config where a designer can confirm it matches the CSS. The same map can be supplied at runtime via the `WEBSHIP_SELECTORS_BREAKPOINTS` env var as JSON, which matters for the next paragraph.

**At scale:** breakpoints are configuration, not test logic, so they belong in `worldParameters` (or the `WEBSHIP_SELECTORS_BREAKPOINTS` env var), never hard-coded as raw pixel sizes inside features. Set it via the env var and CI can run the *same* feature files against a tablet-first profile one night and a desktop-first profile the next, no feature edits -- exactly what you want when a fleet of products share scenario templates but not a design system. Pin the breakpoint map per project in version control so "lg" means one thing in that suite forever; the alternative -- some features at named breakpoints, some at raw pixels -- is how you end up testing a width your CSS never uses and trusting a green run that proved nothing.

---

## Recipe 590: Multi-Viewport Smoke Sweep

### Problem
The hero image looked great on every laptop the team owns and vanished entirely at one specific tablet width, because a media query had an off-by-one boundary. Nobody tested that exact width by hand -- who would? -- and it shipped, and the first you heard was a customer screenshot of an empty hero. You want one test that checks the page holds together at every breakpoint, not just the one on your monitor.

### Solution

```gherkin
  Scenario Outline: Landing renders at <breakpoint>
    Given I am on "/"
    When I set the viewport to the "<breakpoint>" breakpoint
    Then ".hero" should be visible within 1 second
    And there should be no JavaScript errors

    Examples:
      | breakpoint |
      | xs         |
      | md         |
      | lg         |
      | xl         |
```

One outline, four breakpoints, four smoke checks. Catches the bug where the hero disappears at one specific viewport size.

A `Scenario Outline` expands into one real scenario per `Examples` row, so this is four independent runs -- fresh context each, fresh breakpoint each. `set the viewport to the "<breakpoint>"` resolves the name through the breakpoint map from Recipe 589 and resizes the page; `".hero" should be visible within 1 second` is a web-first check that polls for up to a second, so a hero that fades in at one width still passes. The `there should be no JavaScript errors` step leans on webship-js's error capture (`WEBSHIP_JS_ERROR_MODE`, default `warn`) -- it asserts no console errors fired while the page rendered at that size, which catches responsive code that throws only at certain widths. Together they are a real smoke test: the page renders *and* doesn't error, at every breakpoint, from one short outline.

**At scale:** the multi-viewport outline is one of the highest-value patterns you can standardize across a fleet. Template it once -- hero visible, no JS errors, primary nav reachable -- and drop it into every product suite with only the selectors swapped, and you have a comparable responsive-health signal across 20 sites that a PMO can actually read row by row. Mind the CI cost, though: every row is a full scenario, so a five-breakpoint sweep over ten pages is fifty scenarios, and nightly that adds up. The usual split is the full breakpoint sweep on a nightly tag and a two-breakpoint (mobile + desktop) subset on every pull request, so PR feedback stays fast while the deep sweep still runs once a day.

---

## Recipe 591: Cookie + Storage Hybrid Hook

### Problem
The plain clear-everything hook from Recipe 577 works until you hit the one flow that genuinely needs state to survive between scenarios -- a multi-step onboarding split across several scenarios for readability, where each one picks up where the last left off. The blanket hook wipes the state the next scenario needs, so you either delete the hook (and lose isolation everywhere) or fight it. You want a clean slate by default with a documented way to opt out.

### Solution

```js
Before({ tags: 'not @keep-state' }, async function() {
  await this.page.context().clearCookies();
  await this.page.evaluate(() => {
    try { localStorage.clear(); } catch (_) {}
    try { sessionStorage.clear(); } catch (_) {}
  });
});
```

Tag scenarios with `@keep-state` to opt out of the clean slate (rare, but useful for multi-scenario flows).

Two details make this the production-grade version of Recipe 577. First, the `{ tags: 'not @keep-state' }` filter -- Cucumber only runs this `Before` for scenarios that lack the tag, so a `@keep-state` scenario skips the wipe entirely. That is the opt-out: explicit, visible in the feature file, and greppable. Second, the `try/catch` around each `clear()` -- if the page is still on `about:blank` (no origin storage, throws on access, the same trap Recipe 574 and 575 describe), the catch swallows it and the hook moves on instead of failing the run. The cookie clear needs no guard because `context.clearCookies()` is context-level and safe on any page. Remember webship-js already gives every scenario a fresh context, so this is reinforcement plus the opt-out switch, not the only thing standing between you and leaked state.

**At scale:** the tag-gated hook is the shape that survives contact with a real multi-project estate, because it admits the truth that a few flows legitimately need shared state while keeping isolation the default everywhere else. Standardize the tag name -- `@keep-state` -- across all suites so it means the same thing in every repo, and make it a review trigger: a scenario carrying `@keep-state` should have a one-line comment justifying why, because nine times out of ten the real fix is to merge those scenarios into one with a clear sequence of steps rather than relying on cross-scenario order. The `try/catch` matters more here than in a single project: across a fleet, some suites start scenarios with navigation and some don't, and the guard means the *same* hook drops into all of them without one repo's `about:blank` timing breaking the build.

---

## Recipe 592: Selector Inheritance via worldParameters

### Problem
You are onboarding the twentieth project, and once again you are copying the same 20 named selectors -- `main-nav`, `search-box`, `cookie-accept` -- out of the last project's `tests/selectors/`. They have already drifted: three projects spell the cookie button differently, and a platform-wide markup change last month meant 20 separate edits. You want the selectors every site shares to live in one package the whole estate depends on.

### Solution
Publish your selector library as an npm package:

```json
{
  "name": "@yourorg/webship-selectors",
  "version": "1.0.0",
  "main": "selectors.json"
}
```

In each project's `cucumber.js`:

```js
const sharedSelectors = require('@yourorg/webship-selectors');

module.exports = {
  default: {
    worldParameters: {
      selectors: { 
        css: sharedSelectors.css,
        xpath: sharedSelectors.xpath,
      }
    }
  }
};
```

Now every project inherits the same selector vocabulary. Updates to the package propagate via `npm update`.

This works because `worldParameters.selectors` accepts inline `css` and `xpath` maps directly, not only `files`. The registry seeds itself from whatever you put here at world setup -- so requiring the package and spreading its `css`/`xpath` objects into the config means every scenario in the project starts with the shared vocabulary already registered, identical to loading a local JSON file (Recipe 580) except the file is now a versioned dependency. You can still layer a project's own `files:` on top for that product's quirks, and inline `add` steps still override per-scenario; the precedence is the same as everywhere else in the registry. The package's `main` points at the JSON, so `require()` hands you a plain object with `css` and `xpath` keys -- the same shape the registry expects.

**At scale:** this is the recipe the whole chapter has been building toward for a multi-project team. A versioned selector package turns "the login button" from a string copied 20 times into a single source of truth the whole estate consumes -- and `npm`'s semver gives you the release control that copy-paste never could. A platform-wide markup change becomes one PR to the package and a `^1.x` bump that flows out as suites update, instead of 20 hand-edits that miss two repos. Two practices keep it healthy: version it like real code (a selector rename is a minor bump, a removal is a breaking major), and keep the package to genuinely *shared* selectors -- the nav, search, consent, footer that every site has -- while each product's idiosyncratic selectors stay in its own local file. That split is what lets DevOps pin a known-good selector version per environment and lets a PO trust that "search-box" means the same element in every project's report.

---

## Recipe 593: Cookie-Driven Feature-Flag Test

### Problem
The team is rolling out a new checkout behind a `flags.new-checkout` cookie -- on for 10% of traffic, off for the rest. QA needs both branches tested every build, but the old approach was to ask DevOps to flip the flag in a config file, run the suite, flip it back. That is a ticket, a wait, and a flag that occasionally got left in the wrong state. You want one suite that exercises both branches without anyone touching server config.

### Solution

```gherkin
  Scenario: Old checkout
    Given the cookie "flags.new-checkout" is set to "off"
    When I am on "/checkout"
    Then ".checkout-v1" should be visible within 2 seconds

  Scenario: New checkout
    Given the cookie "flags.new-checkout" is set to "on"
    When I am on "/checkout"
    Then ".checkout-v2" should be visible within 2 seconds
```

Two scenarios cover both branches without modifying server-side configuration. Useful during a gradual rollout.

Each scenario plants the flag cookie before navigating -- the Recipe 566 trick -- so the server sees it in the `Cookie` header of the first request and renders the right branch on first paint. This only works because the flag is read from a *cookie*; the value is in the jar before the page loads, no clicking through a settings UI to toggle it. Two separate scenarios, two fresh contexts, so the `off` run can't bleed into the `on` run -- the framework's per-scenario context guarantees each branch starts clean. The web-first `within 2 seconds` on each side absorbs whatever async work the chosen checkout does before its root element appears.

**At scale:** cookie-driven flag testing is how you keep both sides of a rollout green across a fleet without coupling your suite to server state. The pattern that holds: one cookie per flag, both branches tested, and the flag cookie set in the feature rather than in environment config -- so the same suite proves the new and old paths on every environment regardless of what percentage the real flag is dialed to. Tag the new-branch scenarios (`@flag-new-checkout`) so DevOps can include or exclude in-flight features per environment, and so the day the flag is removed you can grep the tag to find every scenario that needs cleaning up. The alternative -- testing only whichever branch the environment's flag happens to expose -- means your green run silently stops covering the other half the moment someone changes a percentage.

---

## Recipe 594: Storage-Driven Multi-Step Wizard Test

### Problem
A five-step signup wizard saves progress to localStorage so users can leave and come back. The "returns mid-wizard" path is exactly where bugs hide -- a restored field that doesn't repopulate, a step that re-asks a question -- but the only way the team tested it was to fill steps 1 through 3 every run, three minutes of setup to reach the ten seconds that mattered, and the setup itself broke whenever step 2's form changed. You want to land on step 3 with real data already in place.

### Solution

```gherkin
  Scenario: Wizard restores from saved state
    Given the local storage "wizard.step" is set to "3"
    And the local storage "wizard.data" is set to "{\"name\":\"Sara\",\"email\":\"sara@example.com\"}"
    When I am on "/wizard"
    Then ".wizard-step-3" should be visible within 2 seconds
    And "input[name=name]" should have value "Sara"
    And "input[name=email]" should have value "sara@example.com"
```

The seeded storage state makes the wizard think the user is mid-flow. Saves you from running steps 1-3 to test step 3.

This is Recipe 574's storage seeding doing real work, and the JSON line is the detail to study. localStorage only stores strings -- the step does `localStorage.setItem('wizard.data', value)` and the value must already be a string -- so the object is hand-serialized in the Gherkin, with `\"` escaping the inner quotes (Cucumber's `{string}` honours backslash-escaped quotes, so the app reads back valid JSON it can `JSON.parse`). Note the step order looks like seed-then-navigate, but the working sequence is the one Recipe 574 spells out: navigate so the origin exists, the keys land, the wizard reads them on load. The three assertions then confirm the restore actually worked end to end -- the right step is showing *and* the fields repopulated -- which is the whole point, because a wizard that jumps to step 3 with empty fields is its own bug.

**At scale:** seeding storage to jump to a deep state is the single biggest runtime saving you can template across a fleet of wizard-heavy products. Every multi-step flow -- checkout, onboarding, application forms -- has the same shape, and across 20 suites the difference between "fill three steps to test the fourth" and "seed and land" is minutes per scenario multiplied by every run, every night. The discipline that keeps it honest: keep at least one *full* walk-through scenario that actually clicks through every step (so the seeding can't mask a broken step transition), and use the seeded shortcut for the many variations of "what happens when you return at step N". Store the seed payloads as named fixtures rather than inline JSON repeated across scenarios, so a schema change to `wizard.data` is one fixture edit, not a sweep through every feature that hard-codes the blob.

---

## Recipe 595: Asserting Cookie Path/Domain

### Problem
Consent has to be remembered across `www.example.com`, `shop.example.com`, and `help.example.com`, which means the consent cookie must be set on the apex `.example.com` domain. A refactor set it on `www.` instead, so users re-saw the banner on every subdomain and the analytics team flagged a sudden spike in consent prompts. The twelve built-in matchers check name and value, not domain -- so this is the case where you write a small custom step against the cookie object itself.

### Solution
Use the world helper:

```js
Then(/^the cookie "([^"]*)" should be on domain "([^"]*)"$/, async function(name, domain) {
  const cookies = await this.page.context().cookies();
  const found = cookies.find(c => c.name === name);
  if (!found) throw friendly(`No cookie named "${name}".`);
  if (found.domain !== domain) {
    throw friendly(`Cookie "${name}" is on domain "${found.domain}", expected "${domain}".`);
  }
});
```

Then in features:

```gherkin
  Scenario: Consent cookie is domain-wide
    Given I am on "https://www.example.com/"
    When I click "Accept all"
    Then the cookie "consent" should be on domain ".example.com"
```

This reaches for the same `context.cookies()` jar the built-in matchers read, but inspects a field they don't expose -- the `domain`. Every object that call returns carries the full cookie: `name`, `value`, `domain`, `path`, `expires`, `httpOnly`, `secure`, `sameSite`. The built-in twelve matchers only branch on `name` and `value`; when you need to assert *scope* -- domain or path -- a six-line custom step against the cookie object is the clean way to do it, and it sits right alongside the standard matchers since they share the data source. The `friendly()` helper is webship-js's error wrapper; it produces the readable `expected vs. actual` failure the built-ins give you, so a domain mismatch reports `is on domain "www.example.com", expected ".example.com"` rather than an opaque assertion error. The same shape extends to `path` -- swap `domain` for `path` and you have a path-scope assertion.

**At scale:** apex-versus-subdomain cookie scope is a bug that only surfaces in production-shaped, multi-subdomain environments -- it passes every single-host test and breaks the moment real traffic crosses subdomains, which is precisely the kind of regression a fleet needs to catch *before* release, not after. If more than one project asserts cookie scope, promote this custom step into your shared step-definitions package so every suite phrases it identically and review can read it without re-deriving the regex. And echo the built-in matchers' lesson here: assert the *least specific* domain that still encodes the contract -- pin `.example.com` when subdomain-sharing is the requirement, but don't over-assert a `path` the app is free to change.

---

## Recipe 596: IndexedDB Cleanup (Custom)

### Problem
Your PWA caches the user's data in IndexedDB so it works offline, and the offline tests passed individually but flaked when run together -- an earlier scenario's IndexedDB records bled into a later one that expected an empty store, so a "first sync" test found data already there. Webship-js clears cookies and localStorage between scenarios but doesn't touch IndexedDB, and it ships no step for it. You want offline scenarios to start with an empty database.

### Solution
Custom step:

```js
Before({ tags: '@offline' }, async function() {
  await this.page.evaluate(async () => {
    const dbs = await indexedDB.databases();
    await Promise.all(dbs.map(db => new Promise((resolve) => {
      const req = indexedDB.deleteDatabase(db.name);
      req.onsuccess = req.onerror = resolve;
    })));
  });
});
```

Now scenarios tagged `@offline` start with empty IndexedDB.

Here is the gap this fills. The framework's fresh-context guarantee covers cookies and storage, but IndexedDB is a third storage layer and its lifetime is its own -- so it is the one piece of browser state that can survive between scenarios if your app writes to it. The hook runs `indexedDB.databases()` inside the page to enumerate every database for the current origin, then deletes each one, wrapping the delete in a promise because IndexedDB's API is event-based (`onsuccess`/`onerror`) rather than promise-based -- the `resolve` on both events means a delete that errors doesn't hang the hook. Note the `{ tags: '@offline' }` scope: this only runs for offline scenarios, because tearing down IndexedDB has a cost and most scenarios never touch it. Same per-origin rule as localStorage -- it clears the current origin's databases, nothing on another subdomain.

**At scale:** IndexedDB, service workers (next recipe), and the HTTP cache are the three state layers that the standard isolation doesn't reach, and across a fleet of PWAs they are the source of the flakiness that "passes alone, fails in the suite". Bundle the cleanup behind a single capability tag -- `@offline` or `@pwa` -- and put the hook in your shared hooks package so every PWA project gets identical teardown without re-implementing the event-to-promise dance. The tag scoping is the cost control: PWA teardown is heavier than a cookie clear, so a marketing-site suite with no offline behaviour never pays for a layer it doesn't use, while the PWA suites get bulletproof isolation on the scenarios that need it.

---

## Recipe 597: Service Worker Reset

### Problem
A service worker caches the app shell, which is great for returning users and a nightmare for testing the *first* visit -- a registered service worker can serve a stale cached shell, so your "first visit shows the install prompt" test sometimes saw the cached version with the prompt already dismissed. It passed or failed depending on what the previous scenario left registered. You want first-visit scenarios to start with no service worker at all.

### Solution

```js
Before({ tags: '@first-visit' }, async function() {
  await this.page.evaluate(async () => {
    const regs = await navigator.serviceWorker.getRegistrations();
    await Promise.all(regs.map(r => r.unregister()));
  });
});
```

Pair with `Before { @first-visit }` to ensure both the SW and the cookie jar are clean for those scenarios.

Service workers are the second state layer the fresh context doesn't fully reset -- a worker registered in one scenario can still be controlling the page in the next, and it intercepts requests invisibly, which is what makes it the sneakiest source of "works locally, flakes in CI". The hook runs inside the page, asks the browser for every registration on the origin via `navigator.serviceWorker.getRegistrations()`, and calls `unregister()` on each, awaiting all of them so the page is genuinely worker-free before the scenario's first navigation. That ordering is the point: unregister *before* you navigate, so the first request actually hits the network instead of a cached shell the old worker would have served. The `{ tags: '@first-visit' }` scope keeps this off the scenarios that don't care, and the per-origin rule applies as always -- you are clearing workers for the current origin only. The tags compose: put `@first-visit @offline` on one scenario and Cucumber runs both this hook and Recipe 596's IndexedDB reset, giving you a genuinely cold browser -- the only honest way to test install prompts, onboarding, and offline-fallback behaviour. Ship both hooks from the shared package and "truly first visit" means the same thing in every product without a bespoke reset per repo.

---

## Recipe 598: Network Cache Bypass

### Problem
A performance regression test for cold start kept reporting suspiciously fast load times, because the browser was serving assets from cache that a real first-time visitor would download from scratch -- so the test "passed" while a genuine cold start was twice as slow. You want every asset fetched fresh, the way a brand-new visitor's browser would, so the cold-start measurement means something.

### Solution
Configure the context:

```js
this.context = await browser.newContext({
  // No setting needed for cache -- Playwright contexts start with empty cache by default.
  // But you can force bypass per-request:
  extraHTTPHeaders: { 'Cache-Control': 'no-cache' },
});
```

Each scenario gets a fresh context, so the cache is per-scenario by default. The header forces cache bypass on the network layer.

The HTTP cache is the third state layer beyond cookies and storage, and here the framework already helps more than you'd expect: a fresh Playwright browser context starts with an empty cache, and webship-js gives every scenario its own context -- so scenario-to-scenario, the cache is clean without you doing anything. What this recipe adds is bypass *within* a scenario that loads the same assets more than once. Setting `extraHTTPHeaders: { 'Cache-Control': 'no-cache' }` on the context attaches that header to every request the context makes, telling the network layer to revalidate rather than serve from its own cache. That is what keeps a multi-navigation cold-start measurement honest -- the second visit in the same scenario still fetches fresh instead of pulling the first visit's assets from memory.

**At scale:** the cache is the quietest of the three extra state layers, which is exactly why it skews performance numbers without anyone noticing. Across a fleet, performance and cold-start tests are only comparable between projects if they all define "cold" the same way -- empty cache, no service worker (Recipe 597), no IndexedDB (Recipe 596). Treat the trio as one capability: a `@cold-start` tag that wires up all three, shipped from the shared hooks package, so a load-time number from project A means the same thing as one from project B and a PMO can actually compare them. The fresh-context default does the heavy lifting; the explicit header is for the in-scenario re-fetch case, and worth a comment in the feature so the next reader knows why it's there.

---

## Recipe 599: Hybrid Cookie + Selectors Pattern

### Problem
Premium subscribers see a different dashboard, and testing it used to mean a real premium login -- a fixture account, credentials in the feature file, a slow trip through the login form -- just to verify one banner renders. Free and premium had separate feature files that drifted apart, and the premium one broke every time the login flow changed. You want one readable scenario that puts the browser in the premium tier and checks the premium UI, no login, no duplicated file.

### Solution

```gherkin
  @premium
  Scenario: Premium dashboard
    Given the cookie "tier" is set to "premium"
    And I add "premium-banner" selector for "section[data-tier='premium']" css selector
    When I am on "/dashboard"
    Then "premium-banner" should be visible within 2 seconds
```

Cookie sets the state, named selector keeps the assertion readable. The combination scales to dozens of tier-aware scenarios.

This is the chapter's two halves working together. The cookie step (Recipe 566) plants `tier=premium` in the context before navigation, so the server renders the premium dashboard on first paint -- no login, because the tier is carried by a cookie the test controls. The named-selector step (Recipe 578) registers `premium-banner` in the scenario's registry, so the assertion reads as the intent (`"premium-banner" should be visible`) rather than the raw `section[data-tier='premium']`. The web-first `within 2 seconds` polls until the banner appears. State and vocabulary are decoupled: the cookie decides *which* UI renders, the selector names *what* you check -- and because both are per-scenario, the free-tier scenario right next to it sets `tier=free` and registers nothing premium, with zero risk of bleed.

**At scale:** tier-, role-, and flag-aware UIs multiply fast across a fleet -- free/premium/enterprise times a dozen products is a lot of variants -- and the cookie-plus-selector pattern is what keeps that combinatorial space testable. Set the tier with a cookie instead of a login, and a tier change is one line, not a fixture account per tier per environment. Pull the tier selectors into a shared JSON file (Recipe 580) so `premium-banner` means the same element everywhere, and tag by tier (`@premium`, `@enterprise`) so CI can run a single tier's scenarios on demand and reports can break quality down by tier. The anti-pattern this replaces -- a logged-in fixture account per tier, duplicated feature files that drift -- is exactly the maintenance load that sinks a multi-project suite over time.

---

## Recipe 600: The State-Setup Hook Pattern

### Pattern
By the time you have onboarded a few projects, you notice the same three lines of setup at the top of nearly every scenario -- clear state, ensure fixtures, register selectors -- copied with small variations, drifting apart, each one a place a future flake can hide. The fix is to stop repeating it in features and put it in one hook. Most tests need three things at the start:

1. Clear cookies, localStorage, sessionStorage
2. Set up any test-data fixtures
3. Register any feature-specific named selectors

A global Before hook handles all three:

```js
const fs = require('fs');

Before(async function() {
  // 1. Clean state
  await this.page.context().clearCookies();
  await this.page.evaluate(() => { localStorage.clear(); sessionStorage.clear(); });
  
  // 2. Fixtures (idempotent -- only inserts missing rows)
  await this.installFixtures();
  
  // 3. Selectors
  const selectorFiles = ['tests/selectors/global.json', 'tests/selectors/forms.json'];
  for (const file of selectorFiles) {
    if (fs.existsSync(file)) {
      const data = JSON.parse(fs.readFileSync(file, 'utf-8'));
      Object.entries(data.css || {}).forEach(([n, s]) => this.addCssSelector(n, s));
      Object.entries(data.xpath || {}).forEach(([n, s]) => this.addXpathSelector(n, s));
    }
  }
});
```

Every scenario starts identically. No surprises. Test logs are clean. The hook is the foundation -- once it's right, everything else gets easier.

The hook ties the whole chapter together, one numbered concern at a time. Step 1 is Recipe 577's clean-state pair, layered on top of the framework's own fresh-context guarantee for belt-and-braces isolation. Step 2 calls a fixture installer that is *idempotent* on purpose -- it only inserts rows that are missing -- so it is safe to run before every scenario without piling up duplicate data or making one scenario's run depend on another's. Step 3 is Recipe 580's file loading done in code: each file is the `{"css": {}, "xpath": {}}` shape the registry expects, read fresh per scenario so nothing leaks across the boundary, with `fs.existsSync` guarding a missing file so an absent `forms.json` degrades quietly instead of crashing the hook. The result is the property every reliable suite has -- every scenario starts from the exact same known state, which is what lets you write each one as if it runs first.

**At scale:** this hook is the contract that makes a multi-project test estate trustworthy. When the same three-part setup -- clean state, idempotent fixtures, file-loaded selectors -- runs identically in all 20 suites, a red scenario means a real regression, full stop, not a leaked cookie or a fixture that didn't load. Ship the hook itself from a shared package so it can't drift between repos, and let each project vary only the data inside it: its own `selectorFiles` list, its own fixture set. That separation -- shared mechanism, per-project data -- is the through-line of this entire chapter, and it is what turns "we test 20 sites" from a maintenance burden into a single pattern you maintain once and a quality signal a PMO, a PO, and a DevOps lead can all read the same way.

---

The cookie, storage, and selector subsystems aren't glamorous. They're the plumbing. But every flaky test eventually traces back to a leaked cookie, a stale storage entry, or a selector that drifted. Master the plumbing and your suite becomes orders of magnitude more reliable.

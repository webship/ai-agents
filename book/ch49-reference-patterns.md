# Chapter 49: Reference Patterns Library

---

The previous 48 chapters covered each subsystem in depth. This chapter is the **reference library** -- a collection of short, self-contained patterns that span multiple subsystems. Each pattern is one or two recipes. They are not organised by feature but by the kind of problem they solve. Think of this chapter as a long appendix you read recipe-by-recipe rather than cover-to-cover.

Use the table of contents and the index to find what you need. Adopt the patterns that fit your stack. Adapt them when they don't.

Each pattern names the situation it solves, shows the shortest solution that works, and explains what the machinery underneath is doing. Where it changes the picture, an **At scale:** note covers what happens when you run the pattern across many projects at once -- because most teams that need a reference library are not running one suite, they are running twenty.

---

## Bootstrapping Patterns

### Recipe 601: The Two-Line Smoke Boot

**Problem:** The deploy went out at midnight, and the first human to learn the homepage is broken is a customer -- at 7:45 the next morning, on social media. Most outages do not need a 400-scenario suite to detect them. They need two lines that run in every pipeline, every time, and answer the only question that matters before any other test runs: is the site up, and is it healthy?

```gherkin
@smoke
Scenario: Site responds
  Given I am on "/"
  Then there should be no JavaScript errors
```

**How It Works:** `Given I am on "/"` resolves the path against `launchUrl` (the `LAUNCH_URL` env var), navigates, and then `smartSettle` holds the step until in-flight fetch/XHR requests drop to zero, pending timers drain, and the DOM has been quiet for at least 250ms -- so a page that technically responded but is still thrashing does not slip through as a pass. The second line reads the JavaScript error buffer webship-js fills from page-error and console listeners for the whole scenario: one uncaught `TypeError` fails the check even when the page looks fine. Reachability and health are different failures -- a 200 with a broken bundle passes the first line and fails the second, and that distinction is the whole recipe.

**At scale:** Make `@smoke` mean exactly this in every project you run, and wire it into every pipeline on every push. When twenty sites share the convention, "is everything up?" stops being twenty Slack questions and becomes one CI dashboard filter. The cost is seconds per project; the payoff is that nobody has to ask.

### Recipe 602: The Status-Sentinel

**Problem:** The frontend smoke is green, but the mobile app is down -- because the API died and nothing tests the API on its own. Backend health gets discovered by whoever consumes it first, and that is usually not your team. One scenario per critical endpoint closes the gap for the cost of three HTTP requests.

```gherkin
@smoke
Scenario Outline: <endpoint> responds healthy
  When I send a GET request to "<endpoint>"
  Then the API response code should be 200

  Examples:
    | endpoint        |
    | /api/health     |
    | /api/version    |
    | /api/ping       |
```

**How It Works:** The API steps run on axios, not on the browser -- no page, no rendering, no settle logic -- so each probe costs milliseconds. The Scenario Outline expands into one independent, named scenario per row: a dead endpoint fails as `/api/version responds healthy` in the report instead of hiding inside a loop. When the API lives on a different host than the site, set it once with `Given the API base URL is "..."` and keep the table paths relative.

**At scale:** Standardize the endpoint names before you standardize the test -- `/api/health` on every service, not `/healthz` here and `/status` there. Once the convention holds, the cucumber JSON from twenty projects answers "which services are degraded right now?" with one query, which is the first portfolio-level quality signal most delivery managers ever get.

### Recipe 603: The Pre-Test Database Seed

**Problem:** Half your scenarios need articles to exist; the other half could not care less. Seed everything for everyone and the suite crawls. Seed nothing and the listing scenarios pass vacuously on a fresh database -- right up until the new hire clones the project, runs the suite against a clean environment, and spends an afternoon discovering that "no results found" was never supposed to be green.

```js
Before({ tags: '@with-articles' }, async function() {
  await this.api.post('/api/test/seed/articles', { count: 5 });
});
```

**How It Works:** Cucumber's `Before({ tags })` option takes a tag expression and runs the hook only for matching scenarios -- everything else skips the cost entirely. The hook calls a test-only seeding endpoint through the world's API client before the browser does anything, so the data exists before the first navigation. Because the seed is per-scenario rather than per-suite, ordering does not matter and parallel shards never fight over shared fixtures. Scenarios declare their data needs in their tags, which means the feature file tells the whole truth.

**At scale:** Standardize the tag vocabulary and the endpoint shape across projects -- `@with-articles`, `@with-users`, `/api/test/seed/*` -- and guard the seed routes so they only exist outside production. A QA lead who owns twenty suites should be able to read any scenario's tags and know its data requirements without opening a single hook file. That convention is worth more than any individual seed.

---

## Authentication Patterns

### Recipe 604: The Auth Storage State

**Problem:** Two hundred scenarios need an admin, and each one starts with the same four steps: open the login page, type, type, click. That is five seconds of pure overhead per scenario -- fifteen minutes of login theatre per run -- and every one of those logins is a fresh chance for a slow auth service to fail a test that was never about login. Log in once, save the proof, and let every scenario start already inside.

```js
// global-setup.js
await page.goto('/user/login');
await page.fill('#username', 'admin');
await page.fill('#password', 'admin-pass');
await page.click('#submit');
await page.context().storageState({ path: 'tests/auth/admin.json' });
```

Then in a Before hook:

```js
Before({ tags: '@as-admin' }, async function() {
  await this.context.addCookies(require('./tests/auth/admin.json').cookies);
});
```

**How It Works:** `storageState()` serialises everything the browser holds that makes you "logged in" -- cookies plus localStorage -- into one JSON file. The Before hook injects those cookies into the scenario's fresh context through `this.context.addCookies()` before the first navigation, so the very first page load arrives already authenticated. The tag filter means only `@as-admin` scenarios pay even that tiny cost; anonymous scenarios stay anonymous. webship-js 2.0 also ships this as built-in steps -- `auth.steps.js` has save/restore/clear storage-state steps -- so you can run the whole dance from Gherkin without writing the hook yourself. One thing to watch: saved sessions expire. Regenerate the JSON at suite start, not once a sprint, or you will chase phantom 403s.

**At scale:** Never commit those JSON files -- they are credentials. Generate them in CI from secrets (admin user and password as env vars per environment) in a setup job, and give every project the same `tests/auth/` layout so the hook code is copy-paste identical across all twenty repos. When a login flow changes in one product, you fix one global-setup file, not two hundred scenarios.

### Recipe 605: The Role-Based Storage State

**Problem:** The bug that reaches production is rarely "admin can't do X" -- admins can do everything, so admin-only suites stay green while editors, reviewers, and plain readers hit walls nobody tested. The day you find out is the day a real reviewer cannot approve anything. Testing permission boundaries means running the same flows as four different people, and nobody wants four copies of the login dance.

One file per role:

```
tests/auth/
  admin.json
  editor.json
  reviewer.json
  reader.json
```

Tag-driven:

```js
['admin', 'editor', 'reviewer', 'reader'].forEach(role => {
  Before({ tags: `@as-${role}` }, async function() {
    await this.context.addCookies(require(`./tests/auth/${role}.json`).cookies);
  });
});
```

**How It Works:** The `forEach` registers four separate `Before` hooks, one per tag expression, at load time -- Cucumber treats them as four independent hooks, and only the one matching the scenario's tags fires. A scenario tagged `@as-editor` gets exactly the editor's cookies injected into its fresh context; an untagged scenario gets nobody's. The role list lives in one line, so adding a fifth role is a one-word change plus one more JSON file. The real value shows in review: `@as-reviewer` at the top of a scenario documents whose eyes the test looks through, without a single login step in the feature file.

**At scale:** Agree on the role vocabulary once -- `@as-admin`, `@as-editor`, `@as-reviewer` -- and use it in every project, even when the underlying products call the roles different things. A QA lead skimming an unfamiliar suite should never need to ask what `@as-editor` means. It also makes portfolio questions answerable: "show me every scenario we run as an anonymous user" becomes a tag grep across repos instead of an archaeology project.

### Recipe 606: The Multi-Factor Mock

**Problem:** Security turned on MFA for the staging environment -- good for security, fatal for automation. The suite cannot read a phone, and the TOTP secret the team shared in a chat thread will rotate the week everyone forgot the tests depend on it. You still need to test everything that lives behind the login, so mock the one request that checks the second factor and walk through the rest of the flow for real.

```gherkin
  Scenario: Login with mocked MFA
    Given the URL "**/api/mfa/verify" returns the JSON:
      """
      { "verified": true }
      """
    When I am on "/user/login"
    And I fill in "username" with "admin"
    And I fill in "password" with "secret"
    And I press "Log in"
    And I fill in "code" with "000000"
    And I press "Verify"
    Then "/dashboard" should be the current path
```

**How It Works:** The mock step registers a `page.route()` handler against the `**/api/mfa/verify` glob -- the `**` matches any scheme and host, so the same pattern works on localhost and on staging. From that moment, any request the verification form sends to that endpoint never leaves the browser: Playwright fulfils it locally with your JSON, the app reads `{ "verified": true }`, and the flow proceeds. Everything else -- the username/password check, the redirect logic, the session cookie -- is the real application. You are not testing MFA itself here (do that in a dedicated lane with a test TOTP seed, or by hand); you are stopping MFA from blocking the two hundred scenarios that live behind it.

**At scale:** Decide per portfolio where MFA gets mocked and where it stays real, and write the decision down -- one project mocking silently while another maintains a TOTP seed is how a real MFA breakage hides for a month. A shared `@mfa-mocked` tag on these scenarios keeps the trade-off visible in every report.

---

## Layout and Visual Patterns

### Recipe 607: The Sticky-Header Guard

**Problem:** The sticky header worked for a year, then a refactor wrapped it in a parent with `overflow: hidden` -- and it quietly stopped sticking. No error, no broken layout, nothing a smoke test notices: the nav simply scrolls away, and users on long articles lose the menu. CSS regressions like this are invisible to functional tests unless you assert the behaviour itself.

```gherkin
  Scenario: Header stays pinned on scroll
    Given I am on "/long-article"
    When I scroll to bottom of the page
    Then the element ".sticky-header" should be at the top of the viewport
```

**How It Works:** The scroll step drives the page to the bottom, and the viewport assertion reads the element's `boundingBox()` and requires its top edge to still sit at (or near) the top of the viewport after scrolling -- that is the operational definition of "sticky". A plain visibility check would pass either way, which is why this recipe exists. The subtlety worth knowing: `position: sticky` fails silently when any ancestor has `overflow: hidden` -- the CSS still parses, the computed style still says sticky, but the box scrolls away. That is exactly the class of regression this guard catches.

**At scale:** Sticky regressions ride in on shared layout components -- a theme bump, a wrapper added in a design-system release -- so they hit every project on that stack at the same time. Point `.sticky-header` at a named entry in a shared `tests/selectors/` JSON file instead of hardcoding the class in each repo, and the day the markup changes you edit one file, not twenty suites. The check costs a scroll and one box read, so run it once per layout rather than once per page.

### Recipe 608: The Footer-Anchor Test

**Problem:** Short pages are where footers betray you. The contact page has three lines of content, and the footer floats mid-screen with a desert of white space below it -- the kind of bug every visitor notices and no suite catches, because the footer is technically present, technically visible, and technically wrong. It usually ships inside a layout-wrapper refactor, and nobody scrolls a short page again until a customer does.

```gherkin
  Scenario: Footer is at the bottom on short pages
    Given I am on "/contact"
    Then the element "footer" should be displayed within a viewport
```

**How It Works:** `should be displayed within a viewport` asserts the element actually sits inside the visible viewport rectangle -- not merely attached to the DOM or un-hidden by CSS. On a short page with a correct min-height layout (a flex column or grid pinning the footer), the footer lands inside the first viewport without scrolling, so the assertion passes; a footer that collapsed upward or got pushed below a phantom spacer fails it. Pick your shortest real page for this test -- the failure mode only exists where content does not fill the screen.

**At scale:** The footer markup is almost always shared -- one template, every page, every site on the stack -- so this is a once-per-project check, not a per-page sweep. Keep the shortest-page path in the same env-var or config slot you use for other layout checks (a thin 404 or contact page works on most sites) so the same scenario drops into any repo without edits. When it fails, it fails everywhere that shares the template, which is the signal you want.

### Recipe 609: The Print Stylesheet Test

**Problem:** Somebody prints your article -- a lawyer, a professor, someone filing paperwork -- and gets three pages of navigation menus, cookie banners, and a hamburger icon rendered as a stray glyph. Print stylesheets break silently because nobody on the team has printed a page since the stylesheet was written, and the regression usually ships inside an unrelated CSS refactor.

```gherkin
  Scenario: Print view hides nav
    Given I am on "/articles/42"
    When I set the viewport media to "print"
    Then "nav" should be hidden
```

**How It Works:** The media step switches the page's emulated media type to `print` -- the same engine switch the browser makes in print preview, driven by Playwright's `emulateMedia({ media: 'print' })` underneath. Every `@media print` rule applies instantly; no paper involved. The assertion then runs as normal: `"nav" should be hidden` polls the element web-first until the print stylesheet's `display: none` takes effect. The media-emulation step is not a webship-js built-in -- it is a thin custom step wrapping `page.emulateMedia()`, so add it once to your project's step files. In a fuller version, assert the other direction too -- the article body should remain visible -- because the second-most-common print bug is a stylesheet that hides everything.

**At scale:** Print CSS lives in the shared theme, so one custom `set the viewport media` step published in your internal step package covers every site that uses it (see Recipe 672 on sharing steps). Most teams skip print testing entirely until a customer complains -- a single emulated assertion per project, run in the nightly lane, turns "nobody has printed this in two years" into a green check.

### Recipe 610: The High-Contrast Test

**Problem:** A user turns on Windows High Contrast or the macOS "Increase contrast" setting because they cannot read low-contrast text -- and your site ignores them completely. The accessibility team flags it during an audit, weeks after the theme that broke the `prefers-contrast` hook shipped, and now it is a remediation ticket instead of a one-line catch. The preference is real, the OS exposes it, and almost nobody tests that the site responds.

```gherkin
  Scenario: High-contrast preference applies
    Given I am on "/"
    When I emulate "prefers-contrast: more"
    Then "body" should have class "contrast-high" within 1 second
```

**How It Works:** `emulate "prefers-contrast: more"` is a custom step wrapping Playwright's `page.emulateMedia({ contrast: 'more' })` -- it sets the media feature the OS would set, so any `@media (prefers-contrast: more)` rule fires exactly as it would for a real user, no OS toggling required. The assertion then polls web-first (100ms, capped here at `within 1 second`) until the contrast hook adds its class. You are testing one thing: that the site notices the preference and reacts. Whether the resulting palette is *good* is a separate, human judgement -- this guards the wiring.

**At scale:** Media-emulation steps are not built in, so write one parametrised `When I emulate "<feature>"` step that maps to `emulateMedia()` and publish it in your shared step package -- the same step then powers Recipes 611 and 612 too. Across a portfolio, group the contrast, reduced-motion, and dark-mode checks under a `@preferences` tag so one filter answers "which sites respect OS accessibility settings?" without opening a single repo.

### Recipe 611: The Reduced-Motion Test

**Problem:** A visitor with a vestibular disorder sets "reduce motion" so the web stops making them dizzy, lands on your homepage, and gets hit by the full parallax-and-carousel show anyway. For some people that is not an annoyance, it is nausea. The setting exists precisely so software can back off, and a site that ignores it is failing the users who need it most -- silently, because the developers testing it never turn the preference on.

```gherkin
  Scenario: Reduced motion disables animations
    Given I am on "/"
    When I emulate "prefers-reduced-motion: reduce"
    Then "html" should have attribute "data-animations" with value "off"
```

**How It Works:** The emulate step sets the `reduced-motion` media feature through `page.emulateMedia({ reducedMotion: 'reduce' })`, so your `@media (prefers-reduced-motion: reduce)` rules activate as if the OS asked for them. This recipe asserts the JavaScript side of the contract -- a `data-animations="off"` flag the app flips when it reads the preference -- which is what gates the heavy scripted animations CSS alone cannot stop. The attribute assertion polls web-first until the flag lands. Pair it with the CSS side (a transition-duration check) when motion lives in both layers.

### Recipe 612: The Dark-Mode System Preference

**Problem:** Your site has a dark mode and a toggle for it, and everyone tested the toggle. Nobody tested the other path: a visitor whose phone is already in dark mode opening the site for the first time, expecting it to respect what the whole rest of their device is doing. When the system-preference branch breaks -- usually because someone refactored the manual toggle and forgot the auto path -- the first load flashes white at people who set dark mode specifically to avoid that.

```gherkin
  Scenario: System dark mode triggers dark theme
    Given I am on "/"
    When I emulate "prefers-color-scheme: dark"
    Then "html" should have attribute "data-theme" with value "dark"
```

**How It Works:** `emulate "prefers-color-scheme: dark"` calls `page.emulateMedia({ colorScheme: 'dark' })` before navigation, so the very first paint already believes the OS is in dark mode -- which is the case you are actually testing, the cold load with no manual override stored. The assertion confirms the app set `data-theme="dark"` on the root, the hook the stylesheet keys off. Test the manual toggle separately; this one guards the automatic branch that only a real OS preference triggers.

**At scale:** Dark mode is a shared theme feature, so the auto-detection logic is identical across every product on the stack -- and so is the regression when a token refactor breaks it. One emulate step plus this three-line scenario, copied unchanged into each repo, catches it everywhere. Keep the root attribute name (`data-theme`) consistent across products so the same scenario truly is copy-paste, not a per-site rewrite.

---

## Network and Performance Patterns

### Recipe 613: The Slow-Network Smoke

**Problem:** The site is fast on the office fibre and unusable on a train. Your team builds on gigabit, so the page that loads instantly for you spends nine seconds blank for a customer on a congested mobile connection -- and the analytics just show them bouncing. Slow-network failures are invisible to everyone who never tests on a slow network, which on most teams is everyone.

```gherkin
  Scenario: Page renders on slow 3G
    Given the network is throttled to "Slow 3G"
    When I am on "/"
    Then "h1" should be visible within 10 seconds
```

**How It Works:** The throttle step is a custom one driving Playwright's CDP `Network.emulateNetworkConditions` (Chromium) with the standard Slow-3G profile -- roughly 400kbps down and ~400ms of latency -- so the request waterfall behaves like a real degraded link, not a fast one with an artificial pause. The assertion then gives the page a generous `within 10 seconds` budget and polls web-first for the `h1` to become visible. You are not measuring exact load time here; you are proving the page is still *usable* -- the headline actually arrives -- when the network is hostile.

**At scale:** Throttling needs the CDP path, which is Chromium-only, so keep these scenarios in a `@slow-network` lane that runs under `BROWSER=chromium` and skip them on firefox/webkit rather than letting them error. Across products, pick one realistic profile and one generous timeout and reuse them everywhere -- a shared budget makes "is the slow-network experience getting worse?" a comparable signal instead of per-project guesswork.

### Recipe 614: The Offline Smoke

**Problem:** You shipped a progressive web app and promised it works offline -- that was the whole pitch to the client. Then a service-worker change silently breaks the offline cache, and the only way anyone finds out is a user in a tunnel staring at the browser's dinosaur instead of your app. Offline behaviour is the one feature you cannot verify by just using the site normally, because you are always online while you build it.

```gherkin
  Scenario: PWA serves offline
    Given I am on "/"
    When the network is offline
    And I reload the page
    Then ".offline-indicator" should be visible
```

**How It Works:** Load the page online first so the service worker installs and caches its assets -- that is what `Given I am on "/"` does here. Then `the network is offline` (a built-in step) calls `context.setOffline(true)`, cutting the browser off the network exactly as airplane mode would. The reload now has no network to fall back on, so if the service worker is healthy it serves the cached shell and the app's own offline indicator appears; if the cache is broken, the reload fails and the assertion catches it. The order is the recipe: cache, then sever, then reload.

### Recipe 615: The Resource-Budget Guard

**Problem:** Page weight only ever grows. Every sprint adds one more font, one more tracking script, one more hero image nobody compressed, and no single change feels big enough to argue about -- until the homepage is two megabytes and the mobile bounce rate has quietly doubled. Without a number in CI, "the site got heavy" is a feeling, not a failing build, and feelings do not block merges.

```gherkin
  Scenario: Homepage stays under 500KB transfer
    Given I start recording network requests
    When I am on "/"
    Then the total transfer size should be less than 500000
```

**How It Works:** `I start recording network requests` is the built-in step that attaches Playwright response listeners and begins logging every request the page makes. Navigation then loads the homepage under that recorder. `the total transfer size should be less than 500000` is a custom step that sums each recorded response's `transferSize` (bytes over the wire, after compression) and asserts the total stays under your budget. Set the recorder before navigating, or the requests fire before anything is listening and the total reads zero.

**At scale:** A page-weight budget only bites if it is the same number across the portfolio -- pull the limit from an env var (say `PERF_BUDGET_BYTES`) so a heavier marketing site and a lean app each get a sane ceiling without editing the scenario. Feed the recorded totals into your report metadata and you get a weight-over-time trend per project, which is the chart that finally wins the "do we really need a fourth analytics script?" conversation.

### Recipe 616: The Third-Party-Free Smoke

**Problem:** Legal signed a contract: no tracking fires before the visitor consents. Then a marketing manager adds a tag through Google Tag Manager without telling engineering, and now the analytics beacon fires on page one for every EU visitor -- a regulatory breach that lives in a third-party config nobody on your team can see in the codebase. The fine is real; the change that caused it never touched your repo.

```gherkin
  Scenario: Critical pageload is third-party-free
    Given I start recording network requests
    When I am on "/"
    Then no request to "*googletagmanager.com*" should have been made
    And no request to "*facebook.net*" should have been made
    And no request to "*hotjar.com*" should have been made
```

**How It Works:** With the recorder running from the first step, `no request to "..." should have been made` is the built-in negative network assertion -- it checks the recorded request log and fails if any URL matched the glob. The `*...*` pattern catches any host containing that domain, so it does not matter which subdomain or CDN edge the tracker loads from. Because it inspects actual outbound requests rather than your source, it catches trackers injected by tag managers, third-party widgets, or someone's "harmless" embed -- the ones code review never sees.

**At scale:** Keep the blocklist of tracker domains in one shared JSON or env list and loop the assertion over it, so adding a newly-banned vendor is a one-line change that propagates to every consent-gated site at once. Tag these `@privacy` or `@consent`: when a regulator or a client asks "prove nothing tracks before consent," a tagged, green run across the portfolio is the evidence -- far better than a screenshot and a promise.

### Recipe 617: The Time-To-Interactive Guard

**Problem:** The page *looks* loaded -- hero image, headline, the lot -- but tap a button and nothing happens for two more seconds while the JavaScript is still parsing. Users read "looks ready" as "is ready," click, get no response, and click again, and your support inbox fills with "the button is broken." Visual completeness and interactivity are different moments, and the gap between them is where rage-clicks live.

```gherkin
  Scenario: Homepage interactive in under 3 seconds
    Given I start the performance timer
    When I am on "/"
    And I wait until the page is interactive
    Then the elapsed time should be under 3000 milliseconds
```

**How It Works:** These three working steps are custom -- a timer-start, an interactivity wait, and an elapsed-time assertion -- because webship-js's philosophy is to wait for events, not clock time, so it ships no stopwatch of its own. The timer records a start mark; `wait until the page is interactive` resolves when the main thread is free enough to handle input (long tasks have drained); the final step asserts the delta stayed under budget. Note this is deliberately the one place you measure wall-clock time on purpose -- everywhere else, a fixed sleep is a smell. Here the elapsed number *is* the thing under test.

**At scale:** Performance gates are noisy on shared CI runners, where a busy host inflates timings and turns a real budget into a flaky one. Run these on a dedicated, consistently-sized runner, keep the budget in an env var per project, and treat a single breach as a warning, not an instant red -- track the trend across runs instead. A performance number is only a portfolio signal when every project measures it on comparable hardware.

---

## API Contract Patterns

### Recipe 618: The Schema Match

**Problem:** The mobile team built against your `/api/users` response for three weeks. Then a backend dev renamed `full_name` to `name` -- a one-word change, perfectly reasonable -- and shipped it, and now every mobile client crashes on the field that vanished. The API had no test that said "this shape is a promise," so nothing stopped the rename, and the first sign of trouble was a spike in app-store one-star reviews.

```gherkin
  Scenario: User endpoint matches schema
    When I send a GET request to "/api/users/1"
    Then the response should match JSON Schema "fixtures/user.schema.json"
```

**How It Works:** The GET runs on axios -- no browser, milliseconds per call. `the response should match JSON Schema "..."` is a custom step that loads a JSON Schema file and validates the response body against it, failing with the exact path that drifted (`/full_name` missing, `/age` is a string not a number). A schema is stricter and more honest than asserting a handful of fields by hand: it pins types, required keys, and structure in one artefact the API team can treat as the contract. Keep the schema in version control next to the test so a breaking change shows up as a failing diff, not a production incident.

### Recipe 619: The Error-Shape Contract

**Problem:** Everyone tests the happy path. The 200 with the right body is well covered -- but the 404, the 422, the 500 are an afterthought, and they are exactly what a client integrator hits first when something goes wrong. If your errors are inconsistent (`{error: "..."}` on one endpoint, `{message: "..."}` on the next, a raw stack trace on a third), every consumer writes bespoke error handling and your support load triples. The error shape is part of the API, even though nobody treats it that way.

```gherkin
  Scenario: 404 error has the expected shape
    When I send a GET request to "/api/users/9999"
    Then the API response code should be 404
    And the JSON response should have property "error.code"
    And the JSON response should have property "error.message"
    And the JSON response should have property "error.docs_url"
```

**How It Works:** The request deliberately asks for a resource that does not exist, so the API takes its error branch. `the API response code should be 404` confirms the status, then `the JSON response should have property "error.code"` (a custom dotted-path existence check, sibling to the JSON-pointer assertions in the fact sheet) verifies each field the error contract promises -- a machine-readable code, a human message, a link to the docs. Testing the error envelope, not just the status, is what makes the failure path as reliable as the success path.

**At scale:** Errors should be uniform across every service you run, or your clients pay the integration tax forever. Lift this into a Scenario Outline that hits each service's known-404 endpoint and asserts the same three fields, and the table becomes the org-wide error-shape contract. The first time a new service ships with a bare `{message}` instead of the standard envelope, this catches it before any consumer does.

### Recipe 620: The Pagination Contract

**Problem:** Pagination is the API feature that breaks quietly. A refactor drops the `links.next` field, and every client that walks pages by following that link silently stops after page one -- no error, no 500, just half the data quietly missing from a dashboard nobody scrolls past the first screen of. The bug surfaces a month later as "the report totals are wrong," and tracing it back to a missing JSON key takes an afternoon.

```gherkin
  Scenario: Page 2 of articles
    When I send a GET request to "/api/articles?page=2&per_page=10"
    Then the API response code should be 200
    And the JSON response should have property "data"
    And the JSON response should have property "meta.total"
    And the JSON response should have property "links.next"
    And the JSON response should have property "links.prev"
```

**How It Works:** Asking for page 2 specifically (not page 1) forces both navigation links to exist -- a `prev` that points back and a `next` that points forward -- which is the case most likely to break. Each `should have property` is a dotted-path existence check against the response body: `data` carries the page of items, `meta.total` the count clients need to compute page counts, and the `links` pair the cursors they follow. You are pinning the envelope a paginated API depends on, the part that lets a consumer walk every page without guessing URLs.

### Recipe 621: The Idempotent-POST Guard

**Problem:** A customer taps "Pay" once, the network stalls, they tap again -- and your payment endpoint charges them twice. Idempotency keys exist to stop exactly this, but the logic is subtle, easy to break in a refactor, and catastrophic when it fails: double charges mean refunds, chargebacks, and a support thread that starts with a furious customer. This is the test you write before you ever ship money-moving code, and keep forever.

```gherkin
  Scenario: Same idempotency key returns same resource
    When I send a POST request to "/api/payments" with body:
      """
      { "amount": 100, "idempotency_key": "abc-123" }
      """
    Then the API response code should be 201
    When I send a POST request to "/api/payments" with body:
      """
      { "amount": 100, "idempotency_key": "abc-123" }
      """
    Then the API response code should be 200
```

**How It Works:** The body docstrings carry the same `idempotency_key` on both POSTs. The first call should create the resource and return `201 Created`; the second, carrying an identical key, should be recognised as a replay and return `200 OK` pointing at the resource already made -- not a second `201`, and crucially not a second charge. The status difference (`201` then `200`) is the observable proof the server de-duplicated. axios keeps the calls fast, and webship-js's API steps share state within the scenario so both requests run against the same backend in sequence.

**At scale:** Any endpoint that moves money or sends an irreversible action -- payments, orders, emails -- needs this guard, so standardise the key field name and the 201/200 convention across services and reuse the scenario shape everywhere. Tag these `@money` or `@critical`: a double-charge regression is the kind of failure where you want the whole portfolio's payment endpoints checked on every release, not just the service someone happened to touch.

### Recipe 622: The Rate-Limit Header

**Problem:** A partner integrates against your API, gets no rate-limit headers back, and has no idea how close they are to the ceiling -- so they hammer you until they hit a wall of 429s with no warning, then open a ticket blaming your API for "randomly failing." Well-behaved clients back off *if you tell them how much budget is left*. The headers are the contract that makes good behaviour possible, and a middleware reorder can strip them without anyone noticing.

```gherkin
  Scenario: Response carries rate-limit headers
    When I send a GET request to "/api/articles"
    Then the response should contain the header "X-RateLimit-Limit"
    And the response should contain the header "X-RateLimit-Remaining"
    And the response should contain the header "X-RateLimit-Reset"
```

**How It Works:** A plain GET is enough -- the headers ride on every response, no need to actually exhaust the limit. `the response should contain the header "..."` is a custom header-existence assertion that inspects the response headers axios captured: the ceiling (`Limit`), how many calls remain in the window (`Remaining`), and when the window resets (`Reset`). Checking presence here is the point; the values change every request, but a missing header means clients are flying blind, and that is the regression you are guarding against.

**At scale:** Rate-limiting is usually one shared gateway or middleware in front of many services, so when it regresses it regresses for all of them. Run this trio against one representative endpoint per service and you confirm the shared layer is intact everywhere with a handful of millisecond-cheap GETs. Standardise on one header convention (the `X-RateLimit-*` family) across the org so partners learn it once and the test reads identically in every repo.

---

## Form Patterns

### Recipe 623: The Required-Field Sweep

**Problem:** Someone adds a new field to the signup form, marks it required in the markup, and forgets to wire the validation message -- so submitting it blank either fails with a generic error or, worse, lets the empty value through. Required-field validation is tedious to test by hand for every field, so it usually is not, and the gaps show up as malformed records in your database long after signup.

```gherkin
  Scenario Outline: <field> is required
    Given I am on "/signup"
    When I leave "<field>" blank
    And I press "Sign up"
    Then I should see "required" near "<field>"

    Examples:
      | field    |
      | name     |
      | email    |
      | password |
```

**How It Works:** The Scenario Outline expands into one independent scenario per row, so a broken validation on `email` fails as its own named line in the report instead of hiding in a loop. `I leave "<field>" blank` and `I should see "required" near "<field>"` are custom steps -- the first ensures the field is empty, the second scopes the "required" text to that field's region so a generic error elsewhere does not pass it by mistake. `I press "Sign up"` is the built-in button press, which triggers `smartSettle` to let client-side validation render before the assertion polls.

**At scale:** The list of required fields lives in the Examples table, so adding a field to the form is one new row, not a new scenario -- and a QA owning many signup forms reads the table as the spec for "what must be filled in here." When several products share a signup component, keep the table shape identical so the same outline drops into each suite and "did required-field validation break anywhere?" stays a one-query question.

### Recipe 624: The Email-Validation Sweep

**Problem:** Email validation is where teams either do too little or get it subtly wrong. Accept `foo@` and you fill the database with addresses that bounce every newsletter; tighten the regex too far and you reject someone's perfectly valid `name+tag@sub.domain` address and lose the signup. The only way to know which malformed inputs your form actually catches is to throw a table of them at it, and nobody does that by hand for every release.

```gherkin
  Scenario Outline: Email "<value>" is invalid
    Given I am on "/signup"
    When I fill in "email" with "<value>"
    And I press "Sign up"
    Then I should see "valid email" near "email"

    Examples:
      | value          |
      | not-an-email   |
      | @example.com   |
      | foo@           |
      | foo bar        |
```

**How It Works:** Each row is a known-bad address that *should* be rejected; the outline runs them as separate scenarios so you see exactly which malformed value slipped through. `I fill in "email" with "<value>"` is the built-in label-first field step -- it resolves the field by its label, then id/name/placeholder. After the button press triggers validation and `smartSettle` lets the message render, `I should see "valid email" near "email"` (a custom scoped-text step) confirms the error appeared next to the field. Keep a sibling scenario for the *valid* cases too, so a regex that rejects everything cannot pass this by being maximally strict.

### Recipe 625: The Password-Strength Bar

**Problem:** The password-strength meter is a small bit of UX with an outsized job: it nudges users toward passwords that will not get their account taken over. When the scoring logic drifts -- a library upgrade rescoring `password` as "strong," or the colour classes getting renamed in a redesign -- the meter starts lying, telling users a weak password is fine. It is the kind of regression that ships unnoticed because the bar still *looks* like it works.

```gherkin
  Scenario Outline: Password "<value>" has strength "<strength>"
    Given I am on "/signup"
    When I fill in "password" with "<value>"
    Then ".strength-bar" should have class "strength-<strength>"

    Examples:
      | value             | strength |
      | a                 | weak     |
      | password          | medium   |
      | P@ssw0rd!Strong   | strong   |
```

**How It Works:** The Examples table pins specific inputs to the strength class each should produce -- a single character is `weak`, a common word is `medium`, a long mixed string is `strong`. `I fill in "password" with "<value>"` types into the field via the built-in label-first step, then `should have class "strength-<strength>"` polls the meter web-first (the strength class lands on the next render) and checks the expected class is present. The class-based assertion is robust to visual restyling -- it tests the scoring decision, not the pixels -- so a colour change does not break it but a scoring regression does.

### Recipe 626: The CSRF Token Refresh

**Problem:** CSRF protection only works if the token actually rotates. If a caching layer freezes the token -- a reverse proxy caching the form HTML, or a CDN rule that should have excluded the page -- every visitor gets the same token, and the protection quietly becomes theatre. It is a security regression with zero visible symptoms: forms still submit, nothing errors, and only a penetration test (or an attacker) discovers the token never changes.

```gherkin
  Scenario: CSRF token is fresh per request
    Given I am on "/posts/new"
    And I capture the CSRF token from the page
    When I wait for 1 second
    And I reload the page
    Then the CSRF token should be different from the captured value
```

**How It Works:** `I capture the CSRF token from the page` and `the CSRF token should be different from the captured value` are custom steps that read the token (from its hidden input or meta tag), stash it on the world, and later compare. The reload fetches a fresh form; a healthy backend issues a new token each time, so the values differ. The one-second wait is here only to make the two requests provably distinct in time -- note webship-js's own guidance is to wait for events rather than the clock, so this fixed wait is a deliberate, narrow exception for a timing-sensitive security check, not a pattern to copy into ordinary flows.

### Recipe 627: The Autosave Test

**Problem:** A writer spends an hour on a long post, the browser crashes, and the draft is gone -- because the autosave that was supposed to fire every thirty seconds silently stopped working after a refactor. Autosave is the safety net nobody notices until it fails, and testing it the honest way would mean sitting and waiting thirty real seconds per run, which no one will do, so it goes untested until someone loses real work.

```gherkin
  Scenario: Draft autosaves every 30 seconds
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/posts/new"
    When I fill in the WYSIWYG field "Body" with the "Hello"
    And I advance the clock by 30 seconds
    Then ".autosave-status" should have text "Saved" within 1 second
```

**How It Works:** `Given the system time is "..."` installs Playwright's clock control via `page.clock`, freezing time at a fixed instant. `I fill in the WYSIWYG field "Body" with the "Hello"` is the built-in rich-text step that types into the editor's `contenteditable` body rather than a plain input. Then `I advance the clock by 30 seconds` fast-forwards the page's virtual clock -- every `setTimeout` and `setInterval` waiting on that thirty-second mark fires instantly, so the autosave timer triggers without thirty real seconds passing. The status assertion then polls web-first until "Saved" appears. You test a half-minute behaviour in milliseconds.

**At scale:** Clock control turns every time-based feature -- autosave, session timeout, polling refresh, scheduled banners -- into a fast, deterministic test instead of a slow, flaky `wait`. Standardise on `Given the system time is "..."` at the top of these scenarios across products so the same fixed instant makes results reproducible, and tag them `@clock` so the time-dependent suite is easy to find and reason about when a timezone change breaks something.

### Recipe 628: The File-Size Validation

**Problem:** The upload form is supposed to cap files at 5MB. Then the client-side check gets removed in a refactor, or the server limit and the message disagree, and a user drags in a 40MB video -- which either uploads and fills your storage, or fails with a cryptic server error instead of the friendly "too big" message. File-size limits are exactly the kind of boundary that rots silently, because no one uploads a deliberately oversized file as part of normal use.

```gherkin
  Scenario: Upload over 5MB is rejected
    Given I am on "/uploads/new"
    When I attach the file "fixtures/large-file-10mb.bin" to "file"
    And I press "Upload"
    Then I should see "exceeds the maximum size of 5 MB" near "file"
```

**How It Works:** `I attach the file "..." to "file"` is the built-in file-input step -- it sets the chosen file on the `<input type="file">` without opening a native OS dialog, which is why upload tests are scriptable at all. The fixture is deliberately over the limit (a 10MB binary against a 5MB cap). After the upload press and `smartSettle`, `I should see "exceeds the maximum size of 5 MB" near "file"` (a custom scoped-text step) confirms the specific rejection message appeared by the field. Keep the oversized fixture in the repo so the boundary case is reproducible on every machine and in CI, not dependent on a file someone has to generate locally.

---

## Search Patterns

### Recipe 629: The Search-As-You-Type Test

**Problem:** Live search feels magical when it works and maddening when the debounce breaks. Lose the debounce and every keystroke fires a request, hammering the backend and flickering results; set it too long and the results lag behind the user's typing. The timing is the feature, and timing is the hardest thing to test reliably -- a real `wait` makes the test slow and flaky, which is why search-as-you-type so often ships with no test at all.

```gherkin
  Scenario: Search updates after debounce
    Given I am on "/search"
    When I fill in "q" with "play"
    And I advance the clock by 300 ms
    Then ".search-result" should have a count of at least 1 within 2 seconds
```

**How It Works:** Typing into the query field starts the debounce timer the search uses to avoid firing on every keystroke. `I advance the clock by 300 ms` jumps the page's virtual clock past that debounce threshold (clock control must be installed for the page first), so the deferred fetch fires immediately instead of after a real 300ms wait. The count assertion then polls web-first up to two seconds for results to render. You are testing the debounce *and* the result render deterministically -- the clock makes the timing exact rather than a hopeful sleep.

### Recipe 630: The No-Results Test

**Problem:** Search for something that does not exist and the page goes blank -- no message, no suggestion, just an empty void that looks broken. The empty state is the path developers never see, because they always search for things that *are* in their test data, so it ships unstyled or missing entirely and users assume the search itself is broken rather than that their query had no matches.

```gherkin
  Scenario: No-results message shows
    Given I am on "/search"
    When I fill in "q" with "xyzzyqwertasdf"
    And I press "Search"
    Then ".no-results" should be visible within 2 seconds
```

**How It Works:** The query is deliberate gibberish chosen to match nothing, forcing the empty branch. After `I press "Search"` submits and `smartSettle` lets the response render, `should be visible within 2 seconds` polls the no-results element web-first (100ms intervals) until it appears or the budget runs out. The two-second window covers a real search round-trip. The whole point is asserting the empty state is a designed, visible message -- not the absence of results, which a naive test would read as a silent pass.

### Recipe 631: The Search-Suggestion Test

**Problem:** Autocomplete suggestions are what make a search box feel fast, and they break in ways that are easy to miss: the suggestion service times out, the endpoint moves, the dropdown renders empty. Users start typing, see nothing helpful, and type the whole query the slow way -- or give up. Because suggestions appear and vanish quickly, a developer clicking through manually rarely notices when they have stopped showing.

```gherkin
  Scenario: Suggestion appears for partial input
    Given I am on "/"
    When I fill in "q" with "play"
    Then ".suggestions li" should have a count of at least 3 within 1 second
```

**How It Works:** Typing a partial query triggers the suggestion fetch. `should have a count of at least 3 within 1 second` is the web-first count assertion -- it polls the suggestion list every 100ms until at least three items exist or the second elapses, which neatly handles the async gap between typing and the dropdown populating without any manual wait. Asserting "at least 3" rather than an exact number keeps the test stable as the underlying content changes while still proving the suggestion machinery fired and returned useful results.

### Recipe 632: The Highlight Test

**Problem:** Search highlighting -- the bold or yellow mark on the matched term in each result -- is a small touch that tells users *why* a result matched. When it breaks, results look arbitrary: the user searched "playwright," sees a list, and cannot tell at a glance which word triggered each hit. It usually breaks when the highlighting moves server-side to client-side or a sanitiser strips the `<mark>` tags, and it is invisible unless you assert the markup is actually there.

```gherkin
  Scenario: Match highlights in results
    Given I am on "/search?q=playwright"
    Then ".result mark" should contain text "playwright"
```

**How It Works:** Navigating straight to a results URL with the query in it skips the typing and lands on a rendered result set. `.result mark` targets the `<mark>` element search engines use to wrap matched terms inside each result, and `should contain text "playwright"` polls web-first until that highlighted element holds the search term. Testing for the `<mark>` element specifically proves the highlighting is semantic markup -- the thing screen readers and styling both rely on -- not just coincidental bold text that happens to look right.

### Recipe 633: The Search-Result-Count Test

**Problem:** The header says "12 results" and the page shows 9. Users notice instantly -- it reads as a bug even if both numbers are individually defensible -- and it erodes trust in the whole search. The mismatch creeps in when the count comes from one query (a fast `COUNT(*)`) and the list from another (a paginated fetch), and the two drift apart after a caching or pagination change that touched only one of them.

```gherkin
  Scenario: Result count matches actual matches
    Given I am on "/search?q=test"
    Then ".result-count" should have text "12 results"
    And ".result" should have a count of 12
```

**How It Works:** Two assertions cross-check each other. `should have text "12 results"` reads the count label the page renders, and `should have a count of 12` is the web-first DOM-counting assertion that tallies the actual `.result` elements present. When they agree, the displayed number is honest; when they disagree, you have caught the count-versus-list drift -- the exact failure a single assertion would miss. Both poll web-first, so they wait out any async render before comparing.

**At scale:** The displayed-count-versus-actual-count check is a reusable shape for any "N items" UI -- a cart total, a notification badge, a table footer -- not just search. Lift it into a step or a tagged convention so the pattern travels across products, and the "header says one thing, list shows another" class of bug gets a standing guard everywhere it can occur instead of being rediscovered per project.

---

## Navigation Patterns

### Recipe 634: The Breadcrumb Test

**Problem:** Breadcrumbs are how users (and search engines) understand where a deep page sits in the hierarchy. When they break -- a routing refactor reorders the trail, or a level drops out -- navigation gets confusing and your structured-data SEO quietly degrades. Breadcrumbs are easy to overlook in testing because the page still works without them; you only feel the loss when someone four levels deep cannot find their way back up.

```gherkin
  Scenario: Breadcrumb is correct
    Given I am on "/articles/category/playwright/42"
    Then the text "Articles" should appear after the text "Home"
    And the text "playwright" should appear after the text "Articles"
    And the text "Article 42" should appear after the text "playwright"
```

**How It Works:** `the text "X" should appear after the text "Y"` is the built-in ordering assertion -- it checks document source order, confirming each crumb sits after its parent in the trail. Chaining the three assertions pins the full ancestry: Home, then Articles, then the category, then the leaf. Asserting *order* rather than mere presence is what makes this meaningful -- a breadcrumb with all the right labels in the wrong sequence is still broken, and only the ordering check catches that.

### Recipe 635: The Active-Menu-Item Test

**Problem:** The active-state highlight in the nav is the user's "you are here" marker. When it breaks -- the active class lands on the wrong item, or on nothing -- people lose their sense of place in the site, and on a deep section it is genuinely disorienting. It is fragile logic (matching the current path against each link) that quietly breaks on a routing change, and it looks fine on the homepage where most people glance, so it ships broken on the inner pages nobody re-checks.

```gherkin
  Scenario: Active menu item highlights
    Given I am on "/articles"
    Then "nav a[href='/articles']" should have class "active"
    And "nav a[href='/']" should not have class "active"
```

**How It Works:** Two complementary web-first assertions make this airtight. `should have class "active"` confirms the link matching the current path is highlighted; `should not have class "active"` confirms the *home* link is not -- guarding against the classic bug where the active state sticks to multiple items or to the wrong one. Testing both the positive and the negative is the trick: a highlight that turns *everything* active would pass a presence-only check but fails the negative assertion here.

### Recipe 636: The Skip-Link Test

**Problem:** A keyboard user lands on your page and has to Tab through forty navigation links before reaching the actual content -- on every single page. The skip link exists to spare them that, and it is the very first thing a screen-reader user encounters, but because mouse users never see it, it is the easiest accessibility feature to break in a header refactor and the last anyone notices. An accessibility audit flags it; by then it has shipped to every page on the site.

```gherkin
  Scenario: Skip link reaches main
    Given I am on "/"
    When I press the key "Tab"
    Then "a.skip-link" should be focused
    When I press the key "Enter"
    Then "#main" should be focused within 1 second
```

**How It Works:** This walks the exact keyboard journey. `I press the key "Tab"` (built-in keyboard step) moves focus to the first focusable element, which on an accessible page is the skip link -- `should be focused` is the web-first focus assertion confirming it. `I press the key "Enter"` activates it, and the second focus assertion confirms focus actually moved into `#main`, polling web-first because the focus shift can be async. Both halves matter: a skip link that is focusable but does not move focus on Enter is broken, and only checking the destination catches that.

**At scale:** The skip link is shared-header markup, so this is a once-per-project guard that protects every page, and it belongs in the same `@a11y` lane as your axe-core audits (Recipe 642) since axe checks structure but not the keyboard *behaviour* this verifies. Keep the `a.skip-link` and `#main` selectors consistent across products and the same scenario drops into any suite unchanged -- accessibility regressions are exactly the ones you want caught uniformly across a portfolio, not project by project.

### Recipe 637: The Back-Navigation Test

**Problem:** A user scrolls deep into a listing, taps into an article, reads it, hits Back -- and lands at the top of the list instead of where they were, or on a blank page, or somewhere else entirely. Broken back-button behaviour is one of the most common and most infuriating web bugs, especially once a site adds client-side routing or aggressive caching, and it is rarely tested because the happy-path forward navigation is the only thing anyone clicks through.

```gherkin
  Scenario: Back returns to listing
    Given I am on "/articles"
    When I click "Read more" in the "First Article" row
    Then the path should be "/articles/1"
    When I move backward one page
    Then the path should be "/articles"
```

**How It Works:** `I click "Read more" in the "First Article" row` is the built-in row-scoped click -- it finds the row whose text matches "First Article" and clicks the "Read more" within it, so the test does not depend on the link's position. `the path should be "/articles/1"` confirms the forward navigation, then `I move backward one page` is the built-in history-back step (Playwright's `page.goBack()`), and the final assertion confirms the browser returned to the listing path. `smartSettle` runs after each navigation so both path checks read a settled URL, not one mid-transition.

---

## Internationalization Patterns

### Recipe 638: The Language-Switch Test

**Problem:** The site shipped in seven languages, and the language switcher is the one control every non-English user touches first. It breaks in the worst way -- the click lands, the label changes, but the URL never switches locale, so the visitor reads English with an Arabic flag in the corner. Nobody on an English-speaking team notices, because nobody on the team clicks the switcher. The first report comes from a customer who assumed the Arabic site simply did not exist.

```gherkin
  Scenario: Switch to Arabic
    Given I am on "/en"
    When I click "العربية"
    Then the path should match "^/ar"
    And the page language should be "ar"
```

**How It Works:** `When I click "العربية"` resolves the switcher by its visible text -- the native language name, which is exactly what a user sees and clicks -- so the test reads like the user's own action. The click triggers navigation, and `smartSettle` holds until the new locale page has loaded and gone quiet before either assertion runs. `the path should match "^/ar"` polls web-first against the URL until it starts with the Arabic locale prefix; the second assertion confirms the document's own `lang` attribute followed, which is the part screen readers and search engines actually read. Asserting both matters: a switcher that changes the URL but not the `<html lang>` is half-broken, and only checking both catches it.

**At scale:** Drive this with a Scenario Outline over your full locale matrix and the native label for each -- one table row per language, the same scenario body. When several multilingual products share that table shape, "did the German switch break anywhere?" is a single tag query across reports rather than a manual click-through of every site.

### Recipe 639: The RTL Layout Test

**Problem:** Right-to-left is the regression that hides behind a passing suite. The Arabic page loads, the translations are correct, every functional test is green -- and the layout is mirrored wrong, sidebar on the left where it should be on the right, the whole reading order fighting the language. It usually breaks when a developer adds a component with hard-coded `left:` and `right:` properties instead of logical ones, and it lands in production because the only people who would catch it by eye are the people who do not review the Arabic build.

```gherkin
  Scenario: Arabic page is RTL
    Given I am on "/ar"
    Then "html" should have attribute "dir" with value "rtl"
    And "body" should have class "rtl"
```

**How It Works:** Both lines are web-first attribute and class assertions -- they poll the element every 100ms for up to five seconds until the value matches, so a `dir` attribute set late by client-side locale detection still passes without a hand-written wait. They check the two switches the rest of the layout hangs off: `<html dir="rtl">` is what flips the browser's own bidirectional engine, and the `rtl` class on `body` is the hook most CSS frameworks use to swap directional rules. This recipe asserts the *contract* -- the flags are set -- not every mirrored pixel. The flags being right is what makes the CSS work, and it is the cheap, stable thing to guard.

**At scale:** Run the same two-line check on every RTL locale across every multilingual project, keyed to a shared `@rtl` tag. The directional flags are the same regardless of which framework a given site uses, so one convention covers the whole portfolio -- and when a component library ships a regression that drops the `dir` attribute, every RTL suite fails on the same line at the same time, which points straight at the shared dependency.

### Recipe 640: The Currency-Formatting Test

**Problem:** A German customer sees their total as `1,234.56 €` instead of `1.234,56 €` and reads it as one euro and change instead of over a thousand. Currency formatting is locale-specific in ways English-speaking developers forget -- the decimal and thousands separators swap, the symbol moves sides -- and the bug is invisible in the codebase because the number is correct; only its presentation is wrong. It breaks when someone formats with a hard-coded `toFixed(2)` and a literal comma instead of the locale-aware formatter.

```gherkin
  Scenario Outline: Currency formats by locale
    Given I am on "/<locale>/prices"
    Then I should see "<expected>"

    Examples:
      | locale | expected      |
      | en-US  | $1,234.56     |
      | de-DE  | 1.234,56 €    |
      | ar-SA  | 1,234.56 ر.س. |
```

**How It Works:** The Scenario Outline expands to one independent scenario per locale, so a broken German format fails as its own named line in the report -- `Currency formats by locale (de-DE)` -- instead of hiding inside a single multi-locale check. Each scenario navigates to that locale's price page and asserts the exact formatted string is present on the page. The separators are the whole point: `en-US` uses comma-thousands and dot-decimal, `de-DE` reverses both and moves the symbol to the end, `ar-SA` carries the Arabic currency symbol. Pinning the literal expected string is what catches a formatter that silently falls back to the default locale.

**At scale:** Keep one canonical price -- here `1234.56` -- across every locale and every project so the expected column is the only thing that changes per row. The day a shared formatting utility regresses, the failing rows across all your suites point at the same value formatted wrong, and you debug the library once instead of per product.

### Recipe 641: The Date-Locale Test

**Problem:** `05/04/2026` means May 4th to an American and April 5th to everyone else, and that ambiguity has cancelled flights and missed deadlines. Date formatting is the most quietly dangerous locale bug because the wrong format is still a valid date -- nothing errors, nothing looks broken, the reader just understands the wrong day. And you cannot test it reliably against the real clock, because "two days ago" changes every time the suite runs.

```gherkin
  Scenario Outline: Date formats by locale
    Given the system time is "2026-05-14T00:00:00Z"
    And I am on "/<locale>/articles/1"
    Then ".published-date" should have text "<expected>"

    Examples:
      | locale | expected         |
      | en-US  | May 14, 2026     |
      | en-GB  | 14 May 2026      |
      | de-DE  | 14. Mai 2026     |
      | ar     | 14 مايو 2026     |
```

**How It Works:** `Given the system time is "..."` installs `page.clock` and pins the browser's notion of "now" to a fixed instant before navigation, so any relative date logic on the page computes against May 14th every run -- the date assertion becomes deterministic instead of drifting with the calendar. Each outline row then loads that locale's article and checks the rendered date string exactly: month name and order differ per locale (`May 14, 2026` vs `14 May 2026` vs `14. Mai 2026` vs the Arabic month name). Freezing the clock is what makes a date test trustworthy in CI -- without it, the expected column would be wrong tomorrow.

**At scale:** The frozen timestamp is your shared fixture; reuse the same instant across every date test in every project so a failure always means a formatting change, never a calendar change. Pair it with the currency outline from the previous recipe under one `@i18n` tag, and a locale rollout becomes a single suite you can point at any site to answer "does this product format dates correctly for all our markets?"

---

## Accessibility Patterns

### Recipe 642: The AA Audit Sweep

**Problem:** The accessibility audit happens once, before launch, a consultant signs off, and then the site evolves for two years with nobody re-running it. Then a procurement officer asks for a current VPAT, or a regulator does, and you discover that a hundred commits of new components quietly reintroduced the contrast and label failures the original audit cleared. Accessibility is not a milestone you pass once; it is a property that decays with every merge unless something guards it on every run.

```gherkin
  Scenario Outline: <page> passes WCAG AA
    Given I am on "<page>"
    Then the page should pass an accessibility audit at level "AA"

    Examples:
      | page         |
      | /            |
      | /articles    |
      | /search      |
      | /contact     |
```

**How It Works:** `the page should pass an accessibility audit at level "AA"` runs axe-core against the live, fully-rendered DOM -- because the step runs after `smartSettle`, it audits the page in its settled post-JavaScript state, not the initial HTML, which is where dynamically-inserted accessibility bugs live. axe checks the WCAG 2.x A and AA rules and the step fails on any violation at that level, naming the rule and the offending element. The Scenario Outline turns one audit into one named result per page, so a regression on `/search` fails as its own line instead of a single opaque "a11y failed". This is automated coverage of the machine-detectable subset of WCAG -- roughly a third of the criteria -- not a substitute for manual testing, but it never forgets and never gets skipped under deadline.

**At scale:** Keep the page list short and high-traffic per project -- the routes that matter, not every URL -- and standardize the `AA` level across the portfolio so "are we still accessible?" has one answer shape everywhere. Feed the per-page results into your report metadata, and a delivery manager gets a comparable accessibility signal across twenty products without commissioning twenty audits.

**Pro Tip:** Audits run on what is rendered, so seed the page into a realistic state first -- log in, expand the menu, open the cookie banner -- or you only test the empty shell. The violations users hit are usually behind the first interaction.

### Recipe 643: The Critical-Only Audit

**Problem:** You inherit a legacy product with three hundred minor accessibility violations, and a full AA gate would be red on day one and stay red for a year -- which means everyone learns to ignore it, which means it guards nothing. But the checkout still has to work for a screen-reader user, or you are leaving revenue and a lawsuit on the table. You need a gate that is achievable today and still blocks the violations that actually stop people from buying.

```gherkin
  Scenario: No critical accessibility violations
    Given I am on "/checkout"
    Then the page should have no critical accessibility violations
```

**How It Works:** This runs the same axe-core engine as the full audit but fails only on violations axe rates `critical` -- the impact tier reserved for problems that outright block a user, like a form control with no accessible name or a trap that keyboard focus cannot escape. Moderate and minor findings are reported but do not fail the step. That makes it a realistic gate for an imperfect codebase: it holds the line on the failures that break the task while you burn down the long tail separately. On a checkout, "critical" maps almost exactly to "a disabled user cannot complete the purchase."

**At scale:** Use critical-only as the enforced gate on every project's revenue-path and sign-up pages, and run the full AA sweep (previous recipe) as a non-blocking report on the side. The two-tier split -- block on critical, track on the rest -- lets a portfolio with wildly different legacy debt share one accessibility policy without grinding any single team to a halt.

### Recipe 644: The Heading-Hierarchy Test

**Problem:** A screen-reader user navigates a long article by jumping heading to heading -- it is how they skim. When a developer reaches for an `<h4>` because it "looked the right size" and skips `<h2>` and `<h3>`, that navigation breaks: the outline reads as a document with chapters but no sections, and the reader loses the thread. Sighted reviewers never catch it because the page looks fine; the headings are styled, just structured wrong.

```gherkin
  Scenario: Heading structure is valid
    Given I am on "/articles/42"
    Then the heading hierarchy should be valid
    And the page should have exactly one h1
```

**How It Works:** `the heading hierarchy should be valid` is a structural check that walks every `<h1>`-`<h6>` in document order and fails if a level is skipped on the way down -- an `<h2>` followed directly by an `<h4>` is the classic failure. `the page should have exactly one h1` enforces the single top-level heading that defines the page's title for assistive tech; zero h1s or several are both common CMS mistakes. Together they assert the document forms a clean tree, which is precisely what heading-based navigation depends on. This is a check styling cannot reveal -- it reads the tag levels, not the rendered sizes.

**At scale:** Headings are the cheapest structural signal to standardize across a content portfolio, because the rule is identical for every page on every site -- one level at a time, exactly one h1. Run it on a sample of each content type per project; a CMS template that emits a skipped level will fail the same way everywhere, pointing at the shared template rather than the content.

### Recipe 645: The Image-Alt Sweep

**Problem:** The gallery is the page most likely to fail this and the page nobody checks. Editors upload images through the CMS for months, and somewhere in there the "alt text" field gets left blank -- because it is optional in the UI, and deadlines are not. A blind user arrives at a wall of "image, image, image, image" with no idea what any of them show. It is the single most common accessibility violation on the web, and it accrues one forgotten upload at a time.

```gherkin
  Scenario: All images have alt text
    Given I am on "/gallery"
    Then every image should have an alt attribute
```

**How It Works:** The step selects every `<img>` in the settled DOM and fails if any one is missing the `alt` attribute, naming the offenders so you know which images to fix. Worth knowing: a present-but-empty `alt=""` is valid and intentional for purely decorative images -- it tells a screen reader to skip them -- so the check is for the attribute's *presence*, not for non-empty text. The hard part of alt text (is it *good* text?) stays human; this guards the floor, which is that the attribute exists at all. On a content-heavy page that floor is exactly where the failures cluster.

**At scale:** Run this against each content type's representative page across the portfolio, and pair it with an editor-side reminder, because the real fix is upstream in the CMS workflow, not in the test. The sweep is your safety net: when a content team's discipline slips on one project, the gallery scenario goes red before the missing alt text ships.

### Recipe 646: The Form-Label Sweep

**Problem:** A designer replaces visible labels with placeholder text because it looks cleaner -- "Email" sitting greyed inside the box instead of above it. It demos beautifully. Then a screen-reader user tabs into the field and hears "edit text, blank," because a placeholder is not a label. Sign-up is the worst place for this to happen: it is the first form a new user meets, and a field they cannot identify is a field they cannot fill.

```gherkin
  Scenario: All form fields have labels
    Given I am on "/signup"
    Then every form field should have an accessible label
```

**How It Works:** This walks every form control and checks each one has an *accessible name* by any of the valid means -- a `<label for>` association, `aria-label`, `aria-labelledby`, or a wrapping label -- the same name-computation a screen reader performs. It deliberately does not accept `placeholder` as a label, which is exactly the trap this catches: a placeholder-only field passes a casual eyeball and fails this step. A field with no accessible name is a field assistive tech announces as anonymous, and on a sign-up form that is a hard stop.

**At scale:** Label resolution is identical across stacks, so this one scenario shape covers the sign-up, contact, and search forms on every project unchanged. Track which forms it runs against per product, and a single shared design-system regression -- a new input component that drops its label association -- lights up the same failure across every suite at once.

### Recipe 647: The Focus-Ring Test

**Problem:** Early in a project, someone hits `outline: none` to kill the "ugly" blue ring around buttons -- and never puts a replacement back. The design looks crisp. Then a keyboard-only user tabs through the page and has no idea where they are, because the one visual cue that tracks focus is gone. It is the most-deleted accessibility feature on the web, removed for aesthetics by people who navigate with a mouse and never feel the loss.

```gherkin
  Scenario: Focus ring is visible on Tab
    Given I am on "/"
    When I press the key "Tab"
    Then the focused element should have visible focus indicator
```

**How It Works:** `When I press the key "Tab"` moves focus to the first focusable element exactly as a keyboard user would, and `smartSettle` lets any focus-driven style change land before the assertion. `visible focus indicator` is a custom step that reads the focused element's computed style and asserts it carries a non-default `outline` or `box-shadow` -- in other words, that the browser's default ring was either kept or deliberately replaced, not stripped to nothing. This catches the specific regression a visibility check never could: an element that is perfectly visible but gives no signal that it holds focus.

**At scale:** Codify this custom step once in your shared step package (the cross-project sharing pattern later in this chapter) so every suite measures "visible focus" the same way -- otherwise each team invents its own threshold and the results stop being comparable. A single design-token change that zeroes out the focus style then fails the same assertion across every project that imports the package.

---

## Robustness Patterns

### Recipe 648: The Stale-Element Guard

**Problem:** This is the flake that gets a scenario quarantined for no good reason. A dashboard widget refreshes itself -- polls the server, re-renders -- and a test that grabbed a button handle a moment earlier clicks a node that no longer exists in the DOM. In a naive framework that throws a "stale element reference" and the test fails maybe one run in ten, always on a different line, which is exactly the profile that makes a team give up and add a retry instead of finding the cause.

```gherkin
  Scenario: Action survives DOM mutation
    Given I am on "/dashboard"
    When I refresh the widget "stats"
    And I click "View details" in the "stats" widget
    Then "/stats" should be the current path
```

**How It Works:** The widget refresh re-renders that part of the DOM, replacing the old nodes. The defence is that webship-js never holds a node reference across steps -- every action re-resolves its locator at the moment it runs, against the live DOM. So `I click "View details"` looks up the button fresh, after the refresh, and clicks whatever node currently matches -- there is no stale handle to go bad. On top of that, the AfterStep `smartSettle` lets the re-render's mutations and any in-flight fetch settle before the click resolves. The recipe proves a stable property: an action that fires *after* a mutation lands on the new element, not a ghost of the old one.

**At scale:** Self-refreshing widgets -- live stats, notification bells, ticker tapes -- are everywhere in modern dashboards, and they are the number-one source of "works alone, flakes in the suite" reports. Make this guard a template across your dashboard-heavy projects so the re-resolution behaviour is tested, not assumed, and so a genuinely stale-prone custom step gets caught here instead of in a 2 a.m. CI run.

### Recipe 649: The Race-Condition Guard

**Problem:** An impatient user double-clicks "Save" -- everyone does it, on a slow connection especially -- and your backend gets two identical POSTs. If the submit button does not disable itself on first click, that is a duplicate order, a double charge, two copies of the same record. It is the bug that does not show up in any single-click test, only in the wild, where real users click the way real users click. The day it bites is the day finance asks why a customer was billed twice.

```gherkin
  Scenario: Two rapid clicks fire only one request
    Given I start recording network requests
    And I am on "/"
    When I click "Save" twice in rapid succession
    Then exactly 1 POST request to "/api/save" should have been made
```

**How It Works:** `Given I start recording network requests` arms webship-js's request recorder, which logs every request the page makes for the rest of the scenario. `click "Save" twice in rapid succession` is a custom step that fires two clicks within 50ms -- deliberately faster than a human but exactly what a fast double-click looks like, and faster than most debounce intervals. The assertion then counts the recorded POSTs to `/api/save` and demands exactly one: proof the UI swallowed the second click (by disabling the button, setting an in-flight flag, or debouncing the handler) before it became a second request. Asserting against the recorded network traffic, not the UI state, is what makes this airtight -- you are checking what actually left the browser.

**At scale:** Double-submit protection is a property every form with a side effect needs and almost none get tested for. Standardize the custom step and the recording pattern across projects so any checkout, any "create" form, any irreversible action carries the same guard. One shared step, audited once, beats discovering per project which forms forgot to disable on submit.

### Recipe 650: The Slow-Render Guard

**Problem:** One slow third-party widget -- a recommendations panel, a social feed, an ads call -- holds the entire homepage hostage because someone awaited it in the critical render path. On a fast office connection nobody notices. On a real mobile connection in a tunnel, the whole page sits blank for five seconds while one non-essential box dawdles. The architecture sin is invisible until the network is slow, which is to say invisible to everyone testing on the office wifi.

```gherkin
  Scenario: Slow widget doesn't block first paint
    Given the URL "/api/slow-widget" is delayed by 5000 ms
    When I am on "/"
    Then "h1" should be visible within 2 seconds
```

**How It Works:** `Given the URL "/api/slow-widget" is delayed by 5000 ms` installs a `page.route()` handler that holds responses to the widget endpoint for five seconds before fulfilling them -- a deterministic slow dependency you control, no real network throttling needed. The assertion then demands the `h1` is visible within two seconds, a window that closes well before the widget can possibly arrive. If the page architecture renders the shell independently and hydrates the widget asynchronously, the header paints fast and the test passes; if the main content is gated behind that one slow call, the two-second budget expires and it fails. The deliberate gap between the 5s delay and the 2s budget is the whole design -- it forces the answer.

**At scale:** Pick the one genuinely-slow-able dependency on each critical page and guard its independence. Across a portfolio this is how you stop "fast on staging, slow in production" surprises -- the failure tells you precisely which project coupled its first paint to a dependency it should have deferred, before a real user on a real phone finds out.

### Recipe 651: The Error-Boundary Test

**Problem:** A backend hiccups and returns a 500 for one widget. The question is whether your dashboard degrades gracefully or detonates. The bad outcome has two faces: a single failed widget takes down the whole page with a white screen of death, or -- nearly as bad -- the widget shows the user a raw `TypeError: Cannot read property 'data' of undefined`, leaking your stack trace and terrifying a non-technical user. A real failure should land as a calm "couldn't load," scoped to the one box that broke.

```gherkin
  Scenario: Component error renders boundary
    Given the URL "/api/widget" returns status 500
    When I am on "/dashboard"
    Then ".widget" should have class "error" within 5 seconds
    And ".widget" should contain text "couldn't load"
    But ".widget" should not contain text "TypeError"
```

**How It Works:** The mock forces the widget's data endpoint to return a 500, simulating a real backend failure deterministically. The three assertions then test the full contract of a good error boundary. `should have class "error"` confirms the boundary actually caught the failure and entered its error state. `should contain text "couldn't load"` confirms the user gets a human message instead of a blank box. And the `But ... should not contain text "TypeError"` is the assertion most teams forget: it proves the raw error did not leak into the UI -- no stack trace, no internal symbol names, nothing that exposes the implementation or alarms the reader. Each is web-first, polling up to its budget while the boundary renders. The boundary catches the error, the user sees a friendly message, and the raw error stays hidden.

**At scale:** Every component that fetches its own data needs a boundary, and "no raw error text leaks to the user" is a portfolio-wide security and trust rule, not a per-widget nicety. Standardize the negative assertion -- never show `TypeError`, `undefined`, a stack frame -- across projects, and you catch the day a refactor strips a boundary and starts spilling internals into the page.

---

## Test-Suite Hygiene Patterns

### Recipe 652: The Flaky-Quarantine

**Problem:** One scenario fails maybe one run in twenty, never the same way twice, and you cannot fix it today. So the team starts hitting "re-run" -- and within a month a red build means nothing, because everyone assumes it is "just the flaky one." That is the real damage of a flaky test: not the test, but the trust it bleeds out of the whole suite. A green build has to mean green, or the suite stops being a gate at all.

```gherkin
@flaky
Scenario: Known flaky scenario
  ...
```

**How It Works:** The `@flaky` tag splits one suite into two lanes by tag expression. The trustworthy lane runs `cucumber-js --tags "not @flaky"` -- it excludes every quarantined scenario, so a failure there is real and blocks the merge. The quarantine lane runs `--tags @flaky --retry 3`, giving the unstable ones up to three attempts and reporting their outcome without gating anything. The point is honesty: the gate stays trustworthy, the flaky tests still run and still produce data, and nobody learns to ignore red. The discipline that makes it work is the audit -- review `@flaky` weekly and either fix the test or delete it, because a quarantine with no exit becomes a graveyard of scenarios nobody trusts and nobody removes.

**At scale:** A flaky-rate-per-project metric, pulled from how many scenarios carry `@flaky` in each repo, is one of the sharpest health signals a QA lead owns -- a suite drifting toward 10% flaky is a suite about to lose its team's trust. Standardize the tag and the two-lane CI split everywhere so the number is comparable, and make "drive `@flaky` back to zero" a tracked goal rather than a someday wish.

### Recipe 653: The WIP Tag

**Problem:** A developer commits a half-written scenario for a feature still in progress -- the steps exist, the feature does not -- and the whole team's pipeline goes red on something that was never supposed to be tested yet. Now everyone is blocked on one person's unfinished work, and the Slack thread starts. Work-in-progress is normal and necessary; it just has no business breaking the shared build.

```gherkin
@wip
Scenario: In progress
  ...
```

**How It Works:** The `@wip` tag carves out a private lane. The default CI profile filters it out -- `--tags "not @wip"` -- so unfinished scenarios never run in the shared pipeline and can never turn it red. The author runs them locally, by tag, while building the feature. When the feature ships, the tag comes off and the scenario joins the real suite in the same commit. The effect is exactly the inverse of the flaky tag: `@flaky` runs everywhere but gates nowhere, `@wip` gates nowhere and runs only locally. Both keep the shared build honest by quarantining a different kind of "not ready."

**At scale:** Add `not @wip` to the default tag filter in every project's shared CI config so the convention is automatic, not something each developer has to remember to honour. Then a `@wip` scenario is safe to commit and push in any repo from day one -- which is what makes developers willing to commit early instead of hoarding work on a branch.

### Recipe 654: The Scenario-Per-Bug

**Problem:** A bug gets fixed, the ticket closes, everyone moves on -- and four months later a refactor quietly reintroduces it, and you fix the exact same bug a second time. Without a test pinning it shut, every closed bug is only closed until someone unknowingly reopens it. The most expensive bugs are the ones you pay to fix twice, and they are entirely preventable with one scenario written at the moment of the fix.

```gherkin
@regression @bug-1284
Scenario: Empty coupon does not crash
  Given I am on "/checkout"
  When I click "Apply" without entering a coupon
  Then I should see "Enter a coupon code"
  And there should be no JavaScript errors
```

**How It Works:** This scenario reproduces the exact path that broke -- empty coupon, click Apply -- and asserts the now-correct behaviour: a polite validation message instead of a crash. The two tags do real work. `@regression` puts it in the slower, comprehensive lane that catches drift over time. `@bug-1284` ties the scenario to the ticket, so when it ever fails again the message points straight at the original bug, the fix, and the discussion -- instant context, no archaeology. The closing `And there should be no JavaScript errors` reads the scenario's JS error buffer (the page-error and console listeners webship-js installs) and fails if the empty-coupon path threw anything, because the original symptom here was an uncaught error, and a "fix" that hides the message but still throws is no fix at all.

**At scale:** Make scenario-per-bug a step in the bug-closing checklist on every project, with the `@bug-<id>` convention shared across repos. The regression suite then becomes a living, executable changelog of every defect the portfolio ever shipped -- and the count of `@bug-*` scenarios per project is a quiet measure of how much hard-won knowledge each suite has captured rather than re-learned.

### Recipe 655: The Smoke-as-Doc Convention

**Problem:** A new engineer joins and asks the question every new engineer asks: "what does this site actually do?" The wiki is eighteen months stale, the original PM has left, and the only fully current description of the product is the code -- which is not something you can hand a newcomer over coffee. So onboarding becomes a fortnight of asking around, and the answers conflict, because everyone remembers a different version of the product.

**How It Works:** Treat the `@smoke` tag as living documentation. If your smoke scenarios are named in plain language -- "User can log in," "Checkout completes," "Search returns results" -- then the list of smoke scenarios *is* the answer to "what does this site do," and it cannot go stale, because it has to pass on every run to stay green.

```bash
cucumber-js --tags @smoke --dry-run
```

`--dry-run` prints every smoke scenario name without executing a single one -- Cucumber resolves and lists them, no browser, instant. The result is a live, code-backed product overview that is current by construction: a feature that no longer works fails its smoke scenario, so a smoke list that is all-green is a feature list that is all-true. That is documentation you can trust precisely because it is enforced.

**At scale:** When every project tags its smoke scenarios with the same plain-language discipline, `--tags @smoke --dry-run` becomes a portfolio catalogue -- run it across repos and you have a current, accurate inventory of what every product does, generated from the suites themselves. For a delivery manager onboarding onto an unfamiliar product, that is the fastest honest overview available anywhere.

### Recipe 656: The "One Failure, One Cause" Test

**Problem:** A scenario fails in CI. You re-run it on its own and it passes. The temptation is overwhelming: tag it `@flaky`, move on, ship. But a test that fails in the suite and passes alone is almost never flaky -- it is telling you something true and uncomfortable, that another scenario left state behind that broke this one. Bury that signal under a retry and you have hidden a real ordering bug that will resurface as a much stranger failure later, when the suite shuffles.

**How It Works:** Before you reach for the flaky tag, run the failing scenario in complete isolation:

```bash
cucumber-js path/to/feature.feature --name "Scenario name"
```

The `--name` filter runs only the scenario whose name matches, nothing before it, nothing after. Now the diagnosis is binary and clean. If it *fails* alone, the bug is in the scenario itself -- fix it. If it *passes* alone but *fails* in the full suite, the scenario is innocent; the fault is inter-scenario contamination -- an earlier scenario created a user, left a cookie, seeded a record, mutated shared data the later one assumed clean. That is a real test-isolation bug, and the fix is making scenarios independent (per-scenario seeding, fresh contexts, cleanup hooks), not papering over it with `--retry`. Investigate before you retry; the retry only hides which scenario is the culprit.

**At scale:** Independence is the property that lets sharding and parallelism work at all -- a suite full of order-dependencies cannot be split across the parallel jobs in the CI patterns later in this chapter. Make "passes in isolation" the standing bar across projects: every scenario must stand alone, and "it's just flaky" must survive this two-command check before anyone is allowed to write it down.

---

## CI Patterns

### Recipe 657: The Sharded Run

**Problem:** The suite grew honestly -- a scenario here, a regression there -- and now it takes forty minutes, and a forty-minute pipeline is a pipeline developers route around. They stop waiting for it, merge on a guess, and the gate you built to catch problems becomes a thing people skip to get their work done. A slow suite does not just cost wall-clock time; it costs the discipline of waiting for green.

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: cucumber-js --shard ${{ matrix.shard }}/4
```

**How It Works:** `--shard 1/4` tells Cucumber to run only its quarter of the scenarios. The CI matrix spins up four jobs in parallel, each running a different shard, so all four quarters execute at once and wall time drops to roughly a quarter -- forty minutes becomes ten. The one prerequisite is the property from the previous recipe: sharding distributes scenarios across machines arbitrarily, so any order-dependency between scenarios will fail unpredictably depending on how the split lands. Independent scenarios are what make horizontal scaling possible at all; this is where that discipline pays off.

**At scale:** Sharding trades money for time -- four jobs cost four runners -- so tune the shard count per project to its suite size rather than maxing it everywhere; a fast suite gains nothing from being split and just burns runners. The portfolio-level move is the same convention across repos so any team can dial shards up as its suite grows, without reinventing the matrix each time.

### Recipe 658: The Browser-Matrix Run

**Problem:** Everything works in Chrome -- because everyone develops in Chrome -- and then a Safari user reports the date picker is dead, or a Firefox user cannot submit the form. The cross-browser bug is the one your team is structurally blind to, because the team's daily browser is the one browser that already works. You cannot afford to run every scenario in every engine on every push, but you cannot afford to find out about WebKit from a customer either.

```yaml
strategy:
  matrix:
    browser: [chromium, firefox, webkit]
steps:
  - run: BROWSER=${{ matrix.browser }} cucumber-js --tags @cross-browser
```

**How It Works:** The `BROWSER` env var selects which engine `playwright.config.ts` launches -- `chromium`, `firefox`, or `webkit` (the open-source engine behind Safari, which is how you test Safari behaviour without owning a Mac in CI). The matrix runs all three in parallel, each picking up its browser from the env var. The `--tags @cross-browser` filter is the cost control: only scenarios you have deliberately marked as browser-sensitive run three times. Everything else runs once, on the default browser, because most logic produces identical results across engines and paying triple for it is waste. You spend the triple cost precisely where engine differences actually bite -- date inputs, CSS layout, media, focus handling.

**At scale:** Curating the `@cross-browser` tag is the real skill here, and it is worth doing once at the portfolio level: a shared checklist of what genuinely differs across engines, applied consistently, so no project either under-tests Safari or wastefully triple-runs its entire suite. That shared judgement is what keeps the cross-browser bill sane across twenty repos.

### Recipe 659: The Always-Upload-Reports Step

**Problem:** A nightly run fails, you open CI the next morning to see why, and there is nothing -- no report, no screenshots, just a red X and a wall of console log. So you re-run it locally hoping to reproduce, which you often cannot, because the failure was environment-specific. The evidence existed for exactly the moment you needed it and then evaporated, because the upload step ran only on success. The one run whose artifacts you actually need is the one run that never kept them.

```yaml
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: cucumber-report
    path: tests/reports/
```

**How It Works:** By default a CI step is skipped once an earlier step in the job fails -- which means a normal upload step never runs when the tests fail, exactly backwards from what you want. `if: always()` overrides that, forcing the upload to run regardless of the test outcome, so the cucumber JSON, the generated HTML report, and the failure screenshots survive the red build and land as a downloadable artifact. The failure case is when the report is most valuable and, without this one line, the case that loses it. It is a one-line change that converts "it failed, no idea why" into "it failed, here is the report and the screenshot of the exact step."

**At scale:** Make `if: always()` non-negotiable in every project's CI template, alongside webship-js's failure screenshots (`WEBSHIP_SCREENSHOT_ON_FAILED` is on by default) and on-failure video (`WEBSHIP_VIDEO=on-failure`). A standing rule that every failed run anywhere in the portfolio leaves behind a complete evidence package -- report, screenshot, video -- is what lets a central QA team triage failures across twenty projects without begging each team to reproduce them by hand.

### Recipe 660: The Test-Run Annotation

**Problem:** The build is red, and to find out which scenario broke a developer has to open CI, click into the job, scroll through hundreds of lines of step output, and read the stack trace. Most will not bother -- they will just re-run and hope, or merge past it. The friction between "the build failed" and "here is the specific thing that failed" is small per incident and enormous in aggregate, and it is exactly the friction that trains people to stop reading their own test results.

```yaml
- name: Annotate failed scenarios
  if: failure()
  run: node scripts/annotate-failures.js
```

**How It Works:** `if: failure()` runs this step only when an earlier step failed -- the mirror of the previous recipe's `always()`. The script `annotate-failures.js` reads `tests/reports/cucumber_report.json`, the machine-readable result webship-js writes for every run, finds the scenarios marked failed, and emits GitHub Actions annotation commands for each. The annotations surface inline on the pull request -- the failed scenario name and message appear right in the PR's checks summary, no drilling into logs required. The cucumber JSON being a stable, documented format is what makes this portable: the same parser works against any project's report because the schema does not change between them.

**At scale:** Write the annotation script once and share it across repos -- the JSON schema is identical everywhere, so one script annotates every project. Better still, point it at your CI provider's native annotation API (GitLab, Azure, and the rest each have one) so the convention holds whatever platform a given team is on. The aggregate payoff is cultural: when every failure names itself on the PR, "read your test results" stops being a request and becomes the path of least resistance.

---

## Local Development Patterns

### Recipe 661: The Headed-Debug Mode

**Problem:** A test fails in CI with "element not found," and you have stared at the selector for ten minutes convinced it is correct. It probably is -- the bug is almost never the selector itself but the timing or the state: a cookie banner covering the button, a modal that opened first, an element that renders a half-second after you looked. Reading the error tells you nothing because the error is a symptom. You need to watch the browser do the thing, slowly, with your own eyes.

```bash
HEADLESS=false SLOW_MO=250 cucumber-js features/login.feature
```

**How It Works:** `HEADLESS=false` tells `playwright.config.ts` to launch a real, visible browser window instead of the invisible headless one CI uses, so you actually see the page. `SLOW_MO=250` inserts a 250ms pause before each Playwright action, slowing the run to human-watchable speed -- without it the browser flashes through every step faster than you can follow. Run a single feature file, not the whole suite, so you are watching the one failure and not waiting through four hundred passing scenarios. Nine times out of ten the cause is obvious the instant you see it: the banner, the wrong tab, the element that was not there yet. This is the first tool to reach for, before any log-reading.

**Pro Tip:** The default `SLOW_MO` is already 300ms in headed mode, so launching headed alone slows things down; bump or drop the value to taste. For a stubborn case, escalate to the inspector in Recipe 663 -- headed mode shows you *what* happens, the inspector lets you stop time and poke at *why*.

### Recipe 662: The Snippet-Generation Mode

**Problem:** You inherit a feature file someone wrote against a different project, or a product owner hands you scenarios they drafted in plain Gherkin. Half the steps have no definition behind them, and finding out which half by running the suite and reading failures one at a time is slow and demoralising. You want the full list of what is undefined up front, before you write a line of glue code.

```bash
cucumber-js --dry-run --format usage-json:snippets.json
```

**How It Works:** `--dry-run` resolves and matches every step against the step definitions without launching a browser or executing anything -- pure parsing, near-instant. The `usage-json:snippets.json` formatter writes a report of which steps matched which definitions and, crucially, generates ready-to-paste snippet stubs for every step that matched nothing. Instead of discovering undefined steps one failed run at a time, you get the complete gap as a single file: here is every step the feature needs that the project does not yet have. Most of those gaps are usually covered by one of webship-js's 411 built-in steps once you adjust the phrasing -- so check the step reference before writing a custom definition you do not need.

**At scale:** When a product owner authors scenarios in shared business language, this is how engineering measures the gap on day one -- run it, count the undefined steps, and you know the implementation cost before committing. Across projects that share a step package, the dry-run also tells you instantly which scenarios a given repo can already run unchanged versus which need new glue, which is exactly the question that decides whether a scenario is portable.

### Recipe 663: The Pause-on-Failure Mode

**Problem:** Headed mode showed you the test fails on a click, but at full speed even slowed to 250ms you cannot tell *why* the locator did not resolve -- the page moves too fast to interrogate. You need to freeze the browser at the exact failing moment and ask it questions: what does the DOM actually look like right now, does my selector match anything, what is covering this element. Watching is no longer enough; you need to stop time and poke.

```bash
PWDEBUG=1 cucumber-js features/login.feature
```

**How It Works:** `PWDEBUG=1` is Playwright's own inspector switch. It forces a headed browser and opens the Playwright Inspector alongside it, pausing execution before each action so the run advances only when you click "step." At any pause you can type a selector into the inspector and watch it highlight -- or fail to highlight -- the matching element on the live page, which answers the locator question definitively instead of by guess. You can open devtools, read the real DOM, and see exactly what state the page is in at the failing step. Where headed mode (Recipe 661) shows you *what* happens, the inspector lets you stop on the failing step and dissect *why*. It is the heaviest debug tool and the one that ends the longest investigations.

**Pro Tip:** `PWDEBUG=1` overrides headless and slow-mo on its own, so you do not need to set them too. Reach for it only after headed mode has not been enough -- it is slower to drive because it is interactive, and most failures confess under Recipe 661 long before you need this.

### Recipe 664: The Single-Step Retry

**Problem:** It is the Tuesday before release, the suite is green except for one scenario that fails intermittently, and you genuinely do not have time to fix the root cause before the cut. You need to ship, and you need a way past the known-flaky scenario that is honest about what it is doing -- not a silent retry baked into every run that quietly masks new failures too.

```bash
cucumber-js --retry 1 features/flaky.feature
```

**How It Works:** `--retry 1` re-runs any scenario that fails, once. If the retry passes, the scenario is reported as passed and flagged as flaky in the output; if it fails both times, it is a genuine failure. Scoping it to a single feature file, not the whole suite, is the entire discipline of this recipe: the retry applies only to the file you have explicitly decided to tolerate, so it cannot hide a fresh failure anywhere else. A blanket `--retry` across the suite is the opposite -- it papers over every flake, old and new, and trains the team to trust a green that re-tried its way there.

**Pro Tip:** This is a tactical bridge to a release, not a fix. The honest path is the quarantine of Recipe 652 -- tag the scenario `@flaky`, retry it only in its own lane, and fix or delete it the same week. A retry left in place for a month is a bug you have agreed to stop seeing.

---

## Patterns That Span Stacks

### Recipe 665: The Drupal Pattern

**Problem:** Your team runs Drupal sites -- a lot of them, if you live anywhere near the Varbase world -- and every one needs the same admin smoke: log in as a privileged user, reach the admin area, confirm it loaded. Writing the login dance by hand for each project is the duplication that the rest of this chapter keeps warning against, and Drupal's user model has its own vocabulary (named users, roles like Webmaster) that generic steps do not speak.

```gherkin
  Scenario: Drupal admin
    Given I am a logged in user with the "Webmaster" user
    And I am on "/admin"
    Then I should see "Administration"
```

**How It Works:** `Given I am a logged in user with the "Webmaster" user` is the Drupal-flavoured authentication step -- it establishes a session for that named user before navigation, the same role-based shortcut the storage-state recipes earlier built by hand, here expressed in Drupal's own terms. The remaining two lines are plain webship-js: navigate to `/admin`, assert the admin landing text is present. The pattern is that the auth step carries the platform-specific knowledge and the rest stays universal, so the scenario reads the same shape whether the stack is Drupal, WordPress, or Laravel -- only the login line changes.

**At scale:** This is the lived reality behind this whole book -- one team maintaining the same admin smoke across an entire distribution's worth of Drupal sites. Standardize the role name (`Webmaster`, or whatever your distribution blesses) and the `/admin` smoke across every Drupal project, and onboarding a new site starts with a scenario you already trust rather than one you write from zero.

### Recipe 666: The WordPress Pattern

**Problem:** Same need, different CMS. The agency runs WordPress sites alongside everything else, and the WordPress admin smoke is identical in intent to the Drupal one -- log in privileged, land on the dashboard, confirm -- but WordPress speaks its own language: roles like `administrator`, the `/wp-admin` path, "Dashboard" as the landing word. A team juggling mixed stacks needs the same scenario shape across all of them, not a different mental model per platform.

```gherkin
  Scenario: WP admin
    Given I am a logged in WordPress user with the "administrator" role
    And I am on "/wp-admin"
    Then I should see "Dashboard"
```

**How It Works:** `Given I am a logged in WordPress user with the "administrator" role` is the WordPress authentication step, establishing a session for that role before navigation -- the WordPress counterpart to the Drupal `Webmaster` step in the previous recipe. Notice the deliberate symmetry: the auth line absorbs everything platform-specific (role vocabulary, session mechanics), and the next two lines -- navigate, assert landing text -- are byte-for-byte the same pattern as Drupal's. The platform difference lives in exactly one step; the rest is portable.

**At scale:** That symmetry is the point for a mixed-stack shop. When the Drupal smoke and the WordPress smoke differ only in the auth line, a QA engineer reads any project's admin scenario at a glance regardless of platform, and the cross-project reports line up cleanly -- "admin reachable" means the same thing and reports the same way whether the site runs WordPress, Drupal, or Laravel.

### Recipe 667: The Laravel Pattern

**Problem:** The custom Laravel application in the portfolio has no CMS role system to lean on -- authentication is by user record, and the protected area you most want to smoke-test is something like Horizon, the queue dashboard that absolutely must stay behind a login. You need the same "reach a protected admin surface" smoke as the CMS projects, expressed in the terms a Laravel app actually uses.

```gherkin
  Scenario: Laravel admin
    Given I am authenticated as user 1
    And I am on "/horizon"
    Then I should see "Horizon"
```

**How It Works:** `Given I am authenticated as user 1` establishes a session for the user with that id before navigation -- Laravel apps identify users by record rather than by CMS role, so the step takes a user id instead of a role name. Same job as the Drupal and WordPress auth steps, different identifier. The remaining two lines are the identical universal pattern: navigate to the protected route, assert its landing text. Across all three stack recipes the structure holds -- one platform-specific auth line, then navigate-and-assert -- which is exactly what makes a single team's scenarios readable across wildly different backends.

**At scale:** For an agency or platform team running Drupal, WordPress, and Laravel side by side, this three-recipe set is the template: pick the auth step that matches the stack, keep everything below it identical. The protected-route smoke then means the same thing in every report regardless of framework, and a new project inherits a working admin scenario the moment you swap in the right login line.

### Recipe 668: The React-SPA Pattern

**Problem:** The whole promise of a single-page app is that navigation feels instant -- the router swaps content client-side, no round trip, no white flash. Then someone wires a nav link as a plain `<a href>` instead of a router link, and clicking "Articles" triggers a full document reload. The URL ends up right and the page looks correct, so every ordinary navigation test passes -- while the app has quietly thrown away the one thing that made it an SPA. Nobody notices until the app feels sluggish and nobody can say why.

```gherkin
  Scenario: React-router navigation
    Given I am on "/"
    When I click "Articles"
    Then the path should be "/articles"
    But no full-page reload should have occurred
```

**How It Works:** The first two assertions are the ordinary navigation check -- click the link, confirm the path changed. The load-bearing line is the `But`: `no full-page reload should have occurred` is a custom step that reads `performance.navigation.type` and asserts it stayed `0` (the "navigate" type set on the initial load) since the scenario began -- meaning the browser never did a hard document load after the first one. A client-side router updates the URL via the History API without a reload, so the type stays `0` and the step passes; a stray `<a href>` forces a real navigation, the type changes, and the step fails. That single assertion is the difference between testing that navigation *works* and testing that it works *the SPA way*.

**At scale:** This is the one assertion that separates an SPA smoke from a generic one -- standardize the custom step across your React, Vue, and similar projects so "navigation stayed client-side" is checked identically everywhere. It is exactly the regression a routine dependency bump or a careless link edit reintroduces, and the kind of thing no manual tester will ever catch by eye.

---

## Closing Patterns

### Recipe 669: The Daily Smoke Run

**Problem:** Not every failure arrives on a commit. The SSL certificate expired at midnight. A nightly cron job corrupted a config. A dependency the site loads from a CDN shipped a breaking change. None of these touch your repo, so per-push CI never fires -- and the first to notice is a customer at 7:45 the next morning. The site can rot overnight while every pipeline sits green, because nothing ran to check.

Schedule a CI job that runs the `@smoke` suite every morning at 6am:

```yaml
on:
  schedule:
    - cron: '0 6 * * *'
```

**How It Works:** The `schedule` trigger with cron `0 6 * * *` runs the workflow at 06:00 every day on a timer, independent of any commit -- so the suite checks the live environment on its own schedule rather than only when someone pushes. Run just the `@smoke` lane: it is fast, it covers "is the site fundamentally working," and it is cheap enough to run unattended every day. The point is to catch the whole class of breakage that lives outside your version control -- expired certs, cron damage, third-party drift -- and have it waiting in your inbox at 6am instead of in a customer's tweet at 7:45.

**At scale:** A 6am smoke across every project, reporting into one place, turns "is the portfolio healthy this morning?" into a dashboard you read with coffee instead of twenty separate checks. Stagger the cron times a few minutes apart so twenty suites do not hammer shared CI runners at the same instant -- a small courtesy that keeps the morning run from becoming its own load spike.

### Recipe 670: The Weekly Full Run

**Problem:** The full suite is too slow and too expensive to run on every push -- if you gate every commit on four hundred scenarios across three browsers, developers wait twenty minutes for feedback and start routing around the gate. So you run a fast subset per commit. But a fast subset, by definition, leaves the long tail of edge-case scenarios unrun for days, and that is exactly where slow drift hides: the rarely-hit path that broke a week ago and nobody noticed.

Schedule the full suite once a week as a comprehensive regression check:

```yaml
on:
  schedule:
    - cron: '0 6 * * 1'   # Mondays 6am
```

**How It Works:** Same `schedule` trigger as the daily smoke, but `0 6 * * 1` fires only on Mondays -- the `1` is the day-of-week field. This is the deliberate counterpart to per-commit CI: fast feedback stays fast (you run the critical subset on every push), and the exhaustive, expensive, slow coverage runs on a cadence the cost can absorb. Monday morning is well chosen -- it surfaces anything that drifted over a quiet weekend before the week's work piles on top of it. The two tiers together give you both quick signal and deep coverage without paying for deep coverage on every commit.

**At scale:** The per-commit-subset plus weekly-full split is the standard cost shape for a portfolio: nightly smoke (Recipe 669) for "is it up," weekly full for "has anything drifted," and the fast lane on every push for "did this change break something." Tune which tier runs how often per project's risk and CI budget, but keep the three-tier vocabulary identical across repos so the cost model and the coverage promise are legible to whoever owns the CI bill.

### Recipe 671: The Pre-Release Sweep

**Problem:** "Are we ready to ship?" gets answered in a meeting, by vibe -- someone asks, a few people nod, the release goes out. There is no shared, written-down, runnable definition of "ready," so the answer depends on who is in the room and how the week went. When a release goes wrong, the post-mortem asks "didn't we test this?" and the honest answer is "we tested some of it, depending."

Tagged set you run before every release:

```bash
cucumber-js --tags "@smoke or @release-gate"
```

**How It Works:** The tag expression `"@smoke or @release-gate"` runs the union of two sets: the smoke scenarios (the site fundamentally works) plus everything explicitly marked `@release-gate` (the specific flows that must pass before a release is allowed out). The result is that "ready to ship" stops being a judgement call and becomes an executable definition -- a green run of this command *is* the readiness criterion, the same one every time regardless of who triggers it. The boolean tag logic is plain Cucumber; the value is entirely in what you choose to tag.

**At scale:** The discipline is curating `@release-gate` jointly with QA and product, per project, and treating that tagged set as the contract for "shippable." Across a portfolio, a passing pre-release sweep becomes the one quality signal a delivery manager can read without translation -- "release X is green against its own gate" means the same thing for every product, which is exactly the comparable, evidence-based sign-off that replaces shipping on trust.

### Recipe 672: The Cross-Project Step Sharing

**Problem:** This is the duplication every preceding "At scale" note has been pointing at. Twenty projects, and each one has its own slightly-different copy of "log in as admin," "wait for the toast," "click the row." You fix a flaky timing bug in one repo's custom step, and the same bug lives on in nineteen others until someone rediscovers it. Copy-paste across repos is the single largest source of wasted effort for a team that runs many suites -- the same fix, paid for again and again.

Publish your project's custom steps as an internal npm package. Import in `cucumber.js`:

```js
require('@yourorg/test-steps');
```

**How It Works:** Cucumber loads step definitions from whatever its config points at, and `require('@yourorg/test-steps')` in `cucumber.js` pulls a published package's steps into the project exactly as if they lived in `tests/step-definitions/`. Now the step logic has one home. Fix a bug or improve a wait once, publish a new version, and every project that bumps the dependency inherits the fix -- no copy-paste, no drift. One source of truth, many consumers. It is precisely the pattern that grew a single team's shared test steps into webship-js itself: a library extracted from running the same suite shape across many sites (Chapter 47 tells that story in full).

**At scale:** This is the keystone of the whole "one team, many projects" theme. Version the shared package, pin it per project, and roll upgrades deliberately so a breaking change does not surprise twenty suites at once. The payoff compounds: every selector convention, every custom assertion, every hard-won timing fix becomes a thing the entire portfolio gets for free the moment it lands in the package -- which is the difference between maintaining twenty suites and maintaining one suite twenty times.

### Recipe 673: The Test-Driven Bug Workflow

**Problem:** The usual bug fix goes: read the report, change some code, click around until it looks fixed, close the ticket. The trouble is you never actually proved you reproduced the original bug -- so you might have fixed a different problem, or fixed nothing and gotten lucky with the click-through. And nothing now stops the bug returning the moment someone refactors near it. "Fixed" by manual confidence is the weakest kind of fixed.

1. Bug reported.
2. Write a scenario that reproduces the bug. Run it. Red.
3. Fix the bug. Run the scenario. Green.
4. Tag with the bug ID. Commit alongside the fix.

**How It Works:** The order is the whole discipline, and step 2 is the one people skip. Writing the failing scenario *first* and watching it go red proves two things at once: that you have genuinely reproduced the reported bug, and that the scenario actually exercises the broken path -- a test that was green before you fixed anything was testing the wrong thing. Then the fix flips it green, which proves the fix addresses *this* bug and not a neighbour. Tagging with the bug id and committing the scenario alongside the fix (the scenario-per-bug pattern from Recipe 654) makes the guard permanent. The bug cannot quietly return, because the scenario that catches it ships in the same commit as the fix and runs forever after. Six months on, that scenario reads as living documentation of what broke and how the fix behaves.

**At scale:** Make "red test before green fix" the standing bug-closing protocol across every project, not a personal habit some engineers keep. The regression suite then grows into an executable history of every defect the portfolio ever shipped and proved fixed -- and "did we ever actually fix bug 1284, and is it still fixed?" becomes a scenario you can run on demand rather than a memory you have to trust.

### Recipe 674: The "Tests as Specs" Conversation

**Problem:** The classic waste loop: a product owner writes acceptance criteria in a doc, QA reads them and rewrites them as test cases, engineering reads the code and builds something subtly different, and the three artefacts drift apart from the first day. By the time anyone notices the mismatch, the feature is half-built against the wrong understanding, and the "did we build the right thing?" argument happens in a release meeting instead of before a line of code was written.

Before writing any new feature, write the acceptance scenarios with the product manager. The scenarios become the spec. Engineering builds against them; QA runs them; product reads them. Three-way alignment from a single artefact.

**How It Works:** Gherkin is plain enough that a product owner can read and even co-author it -- `Given`, `When`, `Then` is structured English, not code. That is the property this recipe exploits. You write the acceptance scenarios *with* the product owner, up front, and that single set of scenarios then serves all three roles at once: it is the spec product signs off on, the target engineering builds against, and the suite QA runs to confirm. There is no translation step to drift through, because there is only one artefact. When the scenarios go green, all three parties are looking at the same proof that the right thing got built -- the alignment is structural, not a matter of everyone remembering the same meeting.

**At scale:** This is where the product owners and BAs in your organisation stop signing off on trust and start signing off on a green run they can read themselves. Across teams, a shared Gherkin vocabulary -- the same step phrasings meaning the same things on every project -- is what lets a product owner read an unfamiliar product's scenarios without a translator, and lets QA reuse a step package (Recipe 672) instead of rebuilding the glue per feature. Spec, build, and acceptance collapsing into one executable artefact is the deepest payoff BDD offers a multi-team organisation.

### Recipe 675: The Long View

**Problem:** Tests get treated as a cost -- a tax on shipping, the thing you cut first when the deadline tightens. Teams that think this way write tests grudgingly, let them rot, and eventually delete the whole suite in frustration when it goes flaky, having gotten almost nothing from the effort they did spend. The misframe is the problem: a suite written as a cost is maintained as a cost and abandoned as a cost.

A test suite is an asset that compounds. Every recipe you add is a fact you no longer have to remember -- knowledge moved out of someone's head, where it leaves with them, and into a check that runs forever. Every regression you catch is a customer who never met the bug. Every smoke check that runs at 6am is a developer who didn't get paged. The value does not stay flat; it accumulates. The scenario you write today guards the code against every future change, including the ones made by people who were not here when you wrote it -- which, across a long-lived portfolio, is most of them.

That compounding is exactly why the "one team, many projects" patterns in this chapter matter more the longer you run them. A shared step package, a tag vocabulary every project honours, a release gate that means the same thing everywhere -- each is an asset that pays a little on every run, on every project, for years. The cost is paid once; the return arrives daily.

Build for the long term. Cover what matters. Delete what doesn't -- a suite full of low-value scenarios is a maintenance cost masquerading as coverage, and pruning is as much a part of the craft as adding. The patterns in this book are the ones that survived a decade of running automated acceptance tests in production, across a distribution's worth of sites maintained by one team. They will, if you let them, do the same for you.

---

That closes the recipe library. Six hundred and seventy-five recipes across forty-nine chapters. Each one is a small tool. Together they form a toolkit large enough to test almost anything a modern web application can throw at you.

Build the test suite you wish you'd inherited.

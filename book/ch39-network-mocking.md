# Chapter 39: Network Mocking and Request Recording

---

A test suite that hits real APIs is at the mercy of real APIs. The payment gateway times out at 3am. The third-party search backend returns a 500 because someone deployed bad config. The stock-data provider returns yesterday's data because a cache hasn't refreshed. None of these are your bugs, but every one of them turns your green build red.

Webship-js ~2.0 ships a **network mocking** family that lets you control exactly what the browser sees over the wire. You can block requests, delay them, force them to return specific status codes, or return arbitrary JSON. Combined with the **network recording** family, you can also assert what requests the page actually fired -- useful for tracking analytics, GTM events, or back-end RPC calls.

This chapter covers the two halves: shaping what the page receives, and observing what the page sends.

---

## When to Mock and When Not To

<!-- diagram: api-flow -->

Mock when:

- The dependency is flaky, slow, or rate-limited
- You need to test error paths (`500`, `503`, network timeout)
- The dependency returns non-deterministic data (timestamps, random IDs)
- You don't control the dependency's environment

Don't mock when:

- The test is an end-to-end smoke test of a real production flow
- You're testing the integration itself, not the page's behaviour
- Stubbing would hide a real bug in your own service

Most browser-side tests fall in the "mock" bucket. Webship-js makes mocking trivial enough that it's the default, not the exception.

The machinery underneath is Playwright's `page.route()`. Every mock step registers a route handler that intercepts matching requests inside the browser's own network stack -- before anything leaves the machine -- and either fulfills them with your canned response, aborts them, or holds them. URL matching uses glob patterns: `*` matches within a path segment, `**` matches across everything, including the scheme and host. That second one is the pattern you will type constantly. `"**/api/stats"` matches `http://localhost:8080/api/stats` on your laptop and `https://staging.example.com/api/stats` in CI, because the `**` swallows the part that differs per environment. A bare `"/api/stats"` matches almost nothing -- get in the habit of leading with `**/`.

---

## Recipe 375: Returning a Mock Status Code

### Problem
A user leaves the dashboard open overnight, comes back with coffee, clicks something -- and the session has long expired. The page should bounce them to `/login`; instead, half the time it renders a broken dashboard skeleton over a wall of console errors. QA cannot reproduce it on demand, because reproducing it honestly means letting a real session age out. The expiry redirect ships untested release after release for exactly that reason.

### Solution

```gherkin
Feature: Session-expiry redirect

  Scenario: 403 on profile triggers login redirect
    Given the URL "**/api/profile" returns status 403 with body ""
    When I am on "/dashboard"
    Then I should be on "/login"
```

The mock registers a `page.route()` handler for the glob `**/api/profile` and fulfills any matching request with a 403 and an empty body -- the request never leaves the browser, no server is involved, no session needs to exist or expire. The leading `**/` is what makes the rule portable: it matches the path regardless of host, so the same scenario works against `localhost` and staging without edits. An empty body is fine here because the page only branches on the status code; when the UI also renders the error payload, give the mock a real body (next recipe). The closing `I should be on "/login"` then verifies the redirect landed.

**At scale:** error paths are the least hand-tested code you own -- nobody manually expires a session on twenty products before a release. Put scenarios like this in a shared error-path feature, tag them `@error-paths`, and run the tag everywhere with the base URL coming from the `LAUNCH_URL` env var. The first full run is illuminating: you will learn which of your products redirect gracefully and which serve a white screen, and your delivery manager finally gets that answer as a table instead of a shrug.

---

## Recipe 376: Returning a Mock Status With Body

### Problem
When a card is declined, the payment API answers `402 Payment Required` with the decline reason in the body, and the checkout page is supposed to surface that reason to the shopper. Testing it against the payment sandbox means maintaining magic test-card numbers that trigger specific declines -- and the sandbox is down or rate-limited the week you need it most. You want the decline path on demand, with the exact message you choose.

### Solution

```gherkin
  Scenario: 402 with message displays card-declined banner
    Given the URL "**/api/payment" returns status 402 with body "Card declined: insufficient funds"
    When I am on "/checkout"
    And I click "Place Order"
    Then ".error-banner" should contain text "insufficient funds" within 3 seconds
```

The route handler fulfills the request with both pieces the page needs: the 402 status that triggers the error branch, and the plain-text body the branch renders. This tests one level deeper than the previous recipe -- not just "the page notices the failure" but "the page shows the user *why*". That distinction catches a classic bug: error handlers that swallow the response body and show a generic "Something went wrong", which is precisely what you do not want a shopper to see when the fix is "use another card". The body here is plain text; when the page parses the error response as JSON, use the docstring form in the next recipe instead.

---

## Recipe 377: Returning a Mock JSON Body

### Problem
The dashboard renders a card per stat from `/api/stats`, and every test against it asserts on numbers that change by the minute. The suite passes at 10am and fails at noon because `activeNow` moved. Then comes the demo where staging had no data and the revenue card proudly displayed zero in front of a client. Live numbers belong in production; tests need numbers that hold still.

### Solution

```gherkin
  Scenario: Dashboard renders mocked stats
    Given the URL "**/api/stats" returns the JSON:
      """
      {
        "totalUsers": 1024,
        "activeNow": 37,
        "revenueToday": 4250.50
      }
      """
    When I am on "/dashboard"
    Then ".stat-users" should have text "1,024" within 2 seconds
    And ".stat-active" should have text "37"
    And ".stat-revenue" should contain text "4,250.50"
```

The docstring body is served back with `Content-Type: application/json` and a 200 status, straight from the route handler -- the page's `fetch` parses it exactly as if the stats service had answered. From the front-end code's point of view there is no way to tell the difference, which is the point: you are testing the *rendering* (does 1024 become "1,024"? does 4250.5 become "4,250.50"?), not the stats service. Deterministic data is what makes exact-text assertions possible; against live data you would be stuck with weak "should be visible" checks that pass even when the formatting breaks.

**At scale:** the payload in the docstring is a contract, and on a multi-project team it pays to treat it like one. Agree on a canonical fixture shape per API and reuse it in every suite that mocks the endpoint -- when the schema gains a field, one review updates the fixtures and you see immediately which projects' UIs care. Deterministic mocks also make products comparable: the same mocked stats rendered by five different dashboards is the cheapest cross-project regression harness you will ever build.

---

## Recipe 378: Blocking a URL Entirely

### Problem
Someone profiles a slow test run and finds the suite spends 800ms of every page load waiting on `analytics.js` from a third-party CDN -- a script whose entire contribution to your tests is making them slower and occasionally flakier when the CDN hiccups. Multiply 800ms by every navigation in every scenario and the analytics vendor is quietly your largest CI line item. You want it gone from test runs without touching the page code.

### Solution

```gherkin
  Scenario: Analytics-free pageload
    Given the URL "https://cdn.example.com/analytics.js" is blocked
    When I am on "/"
    Then "h1" should have text "Welcome" within 2 seconds
```

`is blocked` registers a route that calls `route.abort()` -- the browser sees an immediate network failure instead of a slow round trip. For an async script tag, that failure is a silent no-op; the page carries on without it. The pattern here is an exact URL, which blocks that one file; to silence the vendor wholesale, widen the glob to `"https://cdn.example.com/**"` and every request to that host dies at birth. One warning before you get block-happy: abort a script the page *hard-depends* on and you have not sped up the test, you have changed what it tests -- block the optional, mock the essential.

**At scale:** a shared `Background` that blocks the usual suspects -- analytics, tag managers, chat bubbles, font CDNs, ad slots -- is the single cheapest speedup available to a team running forty suites nightly. It compounds three ways: faster page loads (CI minutes are money), fewer third-party flakes (a CDN outage stops failing your releases), and faster settles (webship-js's smart-wait watches `window.__webshipAjaxCount`, and blocked requests fail instantly instead of dribbling in for seconds). Keep the block list in one shared feature or helper so all projects inherit updates together.

---

## Recipe 379: Delaying a URL

### Problem
The designer spent a sprint on the search loading state -- skeleton, spinner, the works -- and asks whether anyone has actually verified it. Nobody has, because nobody *can*: the staging backend answers in 50ms and the spinner is gone before a human or a screenshot catches it. Loading-state code is real code with real bugs (spinners that never hide are a classic), and it needs a network slow enough to expose it.

### Solution

```gherkin
  Scenario: Spinner is visible during slow request
    Given the URL "**/api/search**" is delayed by 2000 ms
    When I am on "/"
    And I fill in "q" with "playwright"
    And I click the "Search" button
    Then ".spinner" should be visible within 1 second
    And ".spinner" should be hidden within 3 seconds
```

`is delayed by 2000 ms` holds the matched request inside the route handler for two seconds, then lets it continue to the real backend -- this is a delay, not a stub, so the response is still genuine. The trailing `**` on the glob matters here: search requests carry query strings (`/api/search?q=playwright`), and without the wildcard the pattern misses them and the request sails through at full speed. The two assertions then bracket the lifecycle: the spinner must appear well inside the 2-second delay window, and must be gone within a second of the response landing. A spinner that never hides -- the bug users actually report -- fails the second line. Note that the AfterStep smart-settle will not absorb the delay for you: its per-step budget is 1500ms, less than the 2000ms hold, so the explicit web-first assertions are doing the synchronization work here, exactly as they should.

---

## Recipe 380: Asserting a Request Was Made

### Problem
The nastiest front-end failures are the silent ones: a user clicks "Subscribe", the success toast appears, and nothing was ever sent -- a handler got unbound in a refactor, or a `preventDefault` ate the submit. The UI looks perfect; the mailing list stays empty; marketing notices three weeks later. Asserting on what the page *shows* cannot catch this class of bug. You have to assert on what the page *sends*.

### Solution

```gherkin
  Scenario: Subscribe button fires POST
    Given I start recording network requests
    When I am on "/"
    And I fill in "email" with "fan@example.com"
    And I click "Subscribe"
    Then a POST request to "**/api/subscribe" should have been made
```

`I start recording network requests` attaches a listener that logs the method and URL of every request the page fires -- fetch, XHR, scripts, images, beacons -- from that step until the scenario ends. The `Then` scans the log for a POST whose URL matches the glob `**/api/subscribe`. The assertion is method-aware (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`), which is half its value: a subscribe form that fires a GET with the email in the query string is a different bug, and one this assertion also catches. Start the recording *before* the action that should trigger the request -- requests fired before the recording step simply do not exist as far as the log is concerned.

**At scale:** "did the beacon fire" is a question every product team asks about analytics, tag managers, and conversion pixels -- usually in a panic, quarterly. A small `@tracking` feature per project, built on recorded-request assertions, replaces the panic with a nightly green row. It also gives product owners something they have rarely had: readable, self-verifying evidence that the measurement they base decisions on is actually wired up.

---

## Recipe 381: Asserting Any-Method Request

### Problem
Last quarter the health probe was a GET; then someone optimized it to a HEAD to save bandwidth, and the test that asserted `a GET request to "**/api/health"` went red on a change that broke nothing. The behaviour you actually care about is "the page checks its backend on load" -- the verb is an implementation detail that the front-end team is allowed to change without filing a ticket with QA.

### Solution

```gherkin
  Scenario: Health probe runs on load
    Given I start recording network requests
    When I am on "/"
    Then a request to "**/api/health" should have been made
```

Dropping the method from the assertion makes it verb-agnostic: the recorder logs every request's method and URL, and the method-less form matches on URL alone -- GET, HEAD, POST, whatever the probe happens to be this quarter. This is the "test behaviour, not implementation" rule applied to the network layer: pin the verb only when the verb *is* the contract (a POST that creates something), stay loose when it is not (pings, beacons, telemetry). Tests that encode incidental detail are the ones that train teams to ignore red builds.

---

## Recipe 382: Asserting No Request Was Made

### Problem
Your privacy mode promises "no analytics", and legal would like proof rather than promises -- because one stray beacon after a user opts out is not a bug ticket, it is a regulatory incident with a fine attached. The dangerous regression here is invisible by design: the page looks identical whether the tracker fires or not. The only place the truth lives is on the wire, so that is where the assertion has to live.

### Solution

```gherkin
  Scenario: Privacy mode silences analytics
    Given I start recording network requests
    When I am on "/"
    And I click "Enable privacy mode"
    And I click "Subscribe"
    Then no request to "https://analytics.example.com/**" should have been made
```

The negative form scans the same recording and fails if *any* request matches the glob. Pattern width is everything in a negative assertion: trackers append query strings (`/track?event=subscribe&uid=...`), and a too-narrow pattern that misses them makes the test pass while the beacon fires -- a vacuous green, the worst kind. The `/**` suffix closes that gap by matching anything on the host. Ordering matters for the same reason: perform every action that *might* fire the tracker before the `Then`, or you are asserting silence over an empty stretch of the scenario. For belt-and-braces, combine with `is blocked`: the block guarantees no data escapes even if the bug exists, and the recording assertion still fails the build because the *attempt* is logged.

**At scale:** consent and privacy assertions are evidence your PMO and legal team can actually use -- but only if they run everywhere and someone can find the results. Tag them `@compliance`, run the tag across every project nightly, and stamp the report with metadata flags (`--env`, `--app-version`, `--metadata Consent=verified`) so the cross-project dashboard shows opt-out coverage per product, per version. When the regulator letter arrives, "here is last night's run on that exact version" beats any amount of meeting notes.

---

## Recipe 383: Mocking Search Results

### Problem
A search page makes a JSON request to `/api/search?q=...`. You want predictable results for an autocomplete test without worrying about real search-engine quirks.

### Solution

```gherkin
  Scenario: Autocomplete shows top results
    Given the URL "/api/search" returns the JSON:
      """
      {
        "results": [
          {"id": 1, "title": "Playwright in Practice"},
          {"id": 2, "title": "Playing Cucumbers"},
          {"id": 3, "title": "Playgrounds for Developers"}
        ]
      }
      """
    When I am on "/"
    And I fill in "q" with "play"
    Then ".autocomplete .result" should have a count of 3 within 1 second
    And ".autocomplete .result" should contain text "Playwright in Practice"
```

A `q=...` query string variant matches the exact path; webship-js matches against the URL substring by default, so any request whose URL includes `/api/search` is served the mock.

---

## Recipe 384: Combining Block + Mock for Robust Tests

### Problem
Your page calls a primary API and a fallback API. You want to force the fallback path by blocking the primary and mocking the fallback.

### Solution

```gherkin
  Scenario: Fallback path serves response when primary fails
    Given the URL "/api/v2/items" is blocked
    And the URL "/api/v1/items" returns the JSON:
      """
      {"items": [{"id": "legacy", "name": "Legacy Item"}]}
      """
    When I am on "/items"
    Then ".item" should have text "Legacy Item" within 2 seconds
```

Two mock rules, one scenario. The page tries `/api/v2/items`, gets blocked, falls back to `/api/v1/items`, and renders the mocked legacy data.

---

## Recipe 385: Simulating Offline Mode

### Problem
A page should show "You're offline" when the browser detects no network. You want to test the banner without unplugging your laptop.

### Solution

```gherkin
  Scenario: Offline banner appears when network drops
    Given I am on "/"
    When the network is offline
    Then ".offline-banner" should be visible within 2 seconds
    When the network is online
    Then ".offline-banner" should be hidden within 2 seconds
```

`network is offline` flips the browser's `navigator.onLine` flag and fails all requests until you restore connectivity with `network is online`.

---

## Recipe 386: Asserting Request Method Counts

### Problem
A single page-load should fire exactly one POST to `/api/track`. Two POSTs would be a double-firing bug.

### Solution

```gherkin
  Scenario: Tracking fires exactly once per page-load
    Given I start recording network requests
    When I am on "/article/42"
    Then a POST request to "/api/track" should have been made
    When I am on "/article/42"
    Then a POST request to "/api/track" should have been made
```

Each navigation triggers a fresh check. The recording is per-scenario; switching scenarios resets the recorder.

> **Tip:** Network mocking is per-scenario. If you mock `/api/x` in scenario A, scenario B starts with the real URL again. Use a `Background:` block to share mocks across scenarios in a feature.

---

## When Network Mocking Is the Wrong Tool

Mocking is powerful but lossy -- it hides real failure modes. If your test suite never sees the real API, you'll miss schema drifts, auth issues, and rate-limit bugs.

A healthy test pyramid runs:

- **Many unit tests** with full mocking
- **Many BDD browser tests** with selective mocking (mock unstable dependencies, real-test core flows)
- **A small end-to-end smoke suite** with zero mocking, hitting a staging environment

Webship-js can serve both the middle and the smoke layer -- pick the right amount of mocking per suite.

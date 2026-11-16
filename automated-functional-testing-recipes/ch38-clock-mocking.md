# Chapter 38: Clock Mocking and Time Travel

---

Time is the most reliable bug-injector in software. A test that passes Monday at 9am fails Friday at 11pm because the test ran past midnight in another timezone. A reminder fires "two minutes" early because the developer typed 120 instead of 120000. A scheduled job hits the new-year boundary and chooses 1970.

Webship-js ~2.0 adds a full **clock mocking** family. You can freeze the system time at a chosen instant, advance it by milliseconds, seconds, or minutes, and pause or resume the in-page clock. The mock affects `Date.now()`, `new Date()`, `setTimeout`, `setInterval`, and the page's scheduling primitives.

This is the toolkit for testing date-sensitive flows: trial expiry, scheduled emails, "X minutes ago" labels, countdown timers, midnight rollover, leap-year edge cases.

---

## How Clock Mocking Works

<!-- diagram: page-lifecycle -->

Playwright (and therefore webship-js) installs a fake clock in the page context. The fake clock controls every time-reading API the page touches. When you advance the clock, callbacks scheduled with `setTimeout` and `setInterval` fire immediately instead of waiting in real time -- you get a 30-day trial expiry to fire in a 100ms test run.

```
┌────────────────────────────────────────┐
│  Page                                  │
│                                        │
│  Date.now() ──┐                        │
│  new Date()  ─┤                        │
│  setTimeout ──┼──► Fake Clock ◄── Test │
│  setInterval ─┤    (controlled)        │
│  performance ─┘                        │
└────────────────────────────────────────┘
```

Under the hood this is Playwright's `page.clock` API. Installing the fake clock injects a fake-timers implementation into the page *before any page script runs*, replacing `Date`, `setTimeout`, `setInterval`, `requestAnimationFrame`, and `performance.now` with controlled versions. That ordering is why the clock step comes before the navigation step in every recipe in this chapter: install after load, and the page's scripts have already read the real time. Webship-js wraps the whole API as a family of seven steps -- install, advance, pause, resume, and friends -- so the control surface lives in plain Gherkin.

Two control verbs do most of the work:

- **set the system time to "X"** -- jump to an absolute instant
- **advance the clock by N ms/seconds/minutes** -- step forward, firing scheduled callbacks along the way

One interaction worth knowing: webship-js's smart-wait also watches timers, via the `window.__webshipPendingTimers` wrapper. A paused clock can leave callbacks queued forever, but `smartSettle` never hangs on them -- its budget is capped (1500ms after a step, 3000ms per navigation), so the worst a frozen timer costs you is that budget, not a stuck suite.

---

## Recipe 365: Freezing System Time at a Known Instant

### Problem
The sale-countdown test passes all day, then the 11pm nightly run fails because "23 hours" became "22 hours" while the suite queued. Worse, the same test asserts different strings depending on which timezone the CI runner woke up in. Any assertion that contains a number derived from "now" is a coin flip until you pin down what "now" is.

### Solution

```gherkin
Feature: Sale countdown

  Scenario: 24-hour countdown displayed when 24h remain
    Given the system time is "2026-12-25T00:00:00Z"
    And I am on "/sale"
    Then ".countdown" should contain text "23 hours" within 2 seconds
```

`the system time is "X"` accepts any ISO-8601 timestamp and installs the fake clock at that instant -- which is why it comes *before* `I am on "/sale"`. Install happens via an init script, so by the time the countdown code calls `Date.now()`, it already sees December 25th, midnight UTC. The clock is frozen there until you advance it explicitly. Notice the assertion says "23 hours", not "24": with exactly 24 hours remaining, a floor-based formatter displays 23 -- the off-by-one you only catch when you can pin the clock to the boundary. Network requests still happen in real time; only the in-page clock is mocked.

**At scale:** pick one canonical frozen instant for the whole organization -- mid-quarter, midday UTC, nowhere near a DST switch or a month boundary -- and use it as the default in every suite. When twenty projects each invent their own "today", a date-formatting bug reproduces on three of them and the triage call wastes an hour discovering the difference was the fixture, not the code. One shared timestamp makes failures comparable across the portfolio.

---

## Recipe 366: Advancing the Clock by Milliseconds

### Problem
A toast notification auto-dismisses after exactly 3000ms. The first version of this test waited three real seconds and passed -- then someone changed the dismiss delay to 5000ms, and the test kept passing because it asserted "hidden" *eventually*, on a generous timeout. You want a test that fails the moment the product's 3000ms promise changes, and that does not spend three seconds of wall time per run proving it.

### Solution

```gherkin
  Scenario: Toast self-dismisses after 3 seconds
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/"
    When I click "Save changes"
    Then ".toast" should be visible
    When I advance the clock by 3000 ms
    Then ".toast" should be hidden within 1 second
```

The toast's `setTimeout(dismiss, 3000)` sits queued in the fake clock. `I advance the clock by 3000 ms` runs the clock forward through that due time, firing the callback synchronously -- three seconds of product time in roughly a millisecond of wall time. The step ordering carries the logic: assert the toast is visible *before* advancing (proving it showed), advance exactly 3000ms, then assert hidden. If a developer quietly changes the delay to 5000ms, the advance no longer reaches the callback and the test fails -- which is precisely what you want from a test of a timing contract. Multiply the saved wall time across a thousand timing tests and you are buying back hours of CI per night.

---

## Recipe 367: Advancing by Seconds

### Problem
Every login flow with an OTP has one: the "Resend code" button that stays disabled for 60 seconds. Manual QA tests it by sending a code and literally watching the clock -- once per release, per project, sixty seconds of a human's attention each time. And the automated version that waits a real minute is the slowest test in the suite by a factor of ten. The timer logic deserves a test that takes milliseconds.

### Solution

```gherkin
  Scenario: Resend button re-enables after 60 seconds
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/verify"
    When I click "Send code"
    Then "#resend" should be disabled
    When I advance the clock by 60 seconds
    Then "#resend" should be enabled within 1 second
```

The countdown's `setTimeout(..., 60000)` fires instantly when the clock jumps forward 60 seconds -- the fake clock walks through the due time and runs the callback before the step returns. The two web-first assertions around the jump do the real verification: `should be disabled` confirms the lockout engaged, `should be enabled within 1 second` confirms the re-enable handler ran. That trailing `within 1 second` is not for the timer (the timer already fired); it covers the page's own re-render after the callback.

**At scale:** OTP, resend, and rate-limit lockouts appear in every product that has a login -- and they are usually copy-pasted timer code, which means they break in the same way everywhere. Keep one shared feature for the lockout behaviour, drive the base URL per project with the `LAUNCH_URL` env var, and the same five-line scenario verifies the contract on every site you run. When security tightens the lockout from 60 to 90 seconds, the change is one number in one feature file, not an archaeology dig across twenty repos.

---

## Recipe 368: Advancing by Minutes

### Problem
"The editor ate my draft" is the bug report nobody can reproduce, because reproducing it means typing for six minutes and watching whether auto-save actually fires on schedule. The auto-save interval is 2 minutes; the question is not whether it fires once but whether it keeps firing -- interval bugs love to fire the first tick and silently stop. You want consecutive save cycles verified in a test that finishes in under a second.

### Solution

```gherkin
  Scenario: Draft auto-saves every 2 minutes
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/articles/new"
    When I fill in "title" with "Hello"
    And I advance the clock by 2 minutes
    Then ".save-status" should have text "Saved" within 1 second
    When I fill in "title" with "Hello World"
    And I advance the clock by 2 minutes
    Then ".save-status" should have text "Saved" within 1 second
```

Each `advance the clock by 2 minutes` runs the fake clock through exactly one interval tick, firing the auto-save callback once -- not zero times, not twice. That is the property real-time tests cannot pin down: with a real clock, you never know whether your 2-minute wait caught one tick or two. Here the arithmetic is exact. The edit-advance-assert rhythm also catches the "fires once then stops" class of interval bug, because the second cycle is asserted independently of the first. If the second "Saved" never appears, you have found a cleared interval or a debounce that swallowed the tick.

---

## Recipe 369: Pausing the Clock

### Problem
A relative-time label reads "5 seconds ago" when the page renders -- and "6 seconds ago" by the time your assertion runs, because real time kept moving between render and check. The test flakes on slow CI runners and passes locally, which is the most expensive kind of flake to diagnose. You want time to stand completely still while you assert, and to move only when the test says so.

### Solution

```gherkin
  Scenario: Relative-time label is clock-driven
    Given the system time is "2026-01-01T00:00:05Z"
    And the active comment timestamp is "2026-01-01T00:00:00Z"
    And I pause the clock
    When I am on "/comments"
    Then ".comment .relative-time" should have text "5 seconds ago"
    When I advance the clock by 60 seconds
    Then ".comment .relative-time" should contain text "1 minute" within 1 second
```

With the clock paused, `Date.now()` returns the same value no matter how slowly the CI runner grinds through the steps -- the "5 seconds ago" assertion cannot drift into "6 seconds ago", because in-page time literally does not pass between steps. The label then changes *only* in response to `advance the clock`, which is the behaviour under test: the page recomputes relative times from the clock, and the test owns the clock. `pause the clock` is implicit after `set the system time` -- listing it explicitly costs one line and makes the intent obvious to whoever reads the scenario next. One more thing to notice: `the active comment timestamp is ...` is not a webship-js step. It is an app-specific fixture step you write yourself (Chapter 17 territory) to seed the data the page renders -- the clock steps control time, but your fixtures still control state.

---

## Recipe 370: Resuming the Clock

### Problem
You froze time to get the dashboard into a known state -- "yesterday's data, 24 hours stale" -- but the second half of the test clicks "Refresh data" and waits on a real network round trip. A frozen clock and a real network make awkward partners: the request happens in wall time while the page's own timers stay parked. You want deterministic setup, then a clean handover back to real time for the live part.

### Solution

```gherkin
  Scenario: Resume real time for network assertions
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/dashboard"
    When I advance the clock by 24 hours
    And I resume the clock
    And I click "Refresh data"
    Then ".data-loaded" should be visible within 5 seconds
```

Resuming releases the page back to wall time: timers tick on their own again and `Date.now()` moves without your help. One subtlety worth holding onto -- resume does not *rewind*. The page still believes it is 24 hours later than when the scenario started; what changes is that time flows forward from there at the normal rate. So use resume when the remaining steps depend on things actually happening in real time (network latency, animations you want to watch in a recorded video), and stay on the fake clock for everything else. If you find yourself resuming just to "make a wait work", that is usually a smell -- the web-first `within N seconds` assertions poll in wall time regardless, so they work fine under a mocked clock.

---

## Recipe 371: Settling Pending Timers

### Problem
The animations team staggers three callbacks -- 100ms, 500ms, 1000ms -- and next sprint they retune the timings, because that is what animation teams do. The test that advanced the clock by exactly 1000ms breaks on every retune, and the review thread fills with the eternal question: "how long should we advance?" You want a step that means "fire everything that is queued, whatever the delays happen to be this week."

### Solution

```gherkin
  Scenario: All pending timers fire
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/"
    When I click "Start all animations"
    And I wait until pending timers settle
    Then ".animation-done" should have a count of 3 within 1 second
```

`wait until pending timers settle` advances the fake clock just far enough to run every queued callback, in scheduled order, regardless of what the delays are -- fast-forward-everything semantics. It is the clock-mock sibling of the smart-wait machinery: where `smartSettle` *observes* `window.__webshipPendingTimers` and waits for the count to reach zero in real time, this step actively drains the queue by moving time itself. The decision rule between this recipe and the previous three: advance by an exact amount when the delay *is* the contract you are testing (a 60-second lockout must be 60 seconds); settle everything when the delays are incidental (animation polish that retunes every sprint).

**At scale:** adopt that decision rule as a written convention and the "how long should we advance?" review argument disappears across all your projects. Hardcoded advance values in feature files should always trace to a documented product number -- if a reviewer cannot find where the spec says 60 seconds, the scenario should be settling timers instead. That single convention keeps twenty suites from breaking every time a front-end team retunes its easing curves.

---

## Recipe 372: Testing a Daily Cron Boundary

### Problem
The daily quota resets at midnight UTC, which means your users in Auckland hit the rollover during their workday and find the bugs first -- while your developers, asleep nine timezones away, never see the boundary at all. "Wait until tomorrow and check" is not a test plan. You need to stand at 23:59:50 and step across midnight on demand, as many times per day as the suite runs.

### Solution

```gherkin
  Scenario: Quota resets at UTC midnight
    Given the system time is "2026-03-14T23:59:50Z"
    And I am on "/account"
    Then ".quota-remaining" should have text "3 / 5 actions"
    When I advance the clock by 15 seconds
    Then ".quota-remaining" should have text "5 / 5 actions" within 1 second
```

Freeze ten seconds before midnight, assert the old day's state, then cross 00:00:00 in a single 15-second advance. Any interval the page runs ticks during that advance, reads the new date from the mocked `Date`, and refreshes the counter -- the same code path a real user's open tab would execute at midnight, compressed into milliseconds. The first assertion is load-bearing: it proves the *pre*-rollover state, so when the second assertion passes you know the change happened because of the boundary, not because the page rendered fresh.

**At scale:** date boundaries are a category, not a one-off -- midnight rollover, month-end, December 31st, February 29th, the DST switches. Keep a small "date edges" feature per product that pins each gnarly instant with a frozen clock, and run it nightly everywhere. It is a few dozen lines per project, and it converts "we think the leap-year path works" into a green row on the cross-project report -- the difference between a guess and an answer when someone asks in February.

---

## Recipe 373: Testing a Trial Expiry Banner

### Problem
Trial expiry is checked on page load: "expires in 1 day" today, "Trial expired" tomorrow. The support tickets cluster exactly at that boundary -- users who swear they had a day left, billing flows that locked a day early. QA cannot verify the fix without either waiting out a real seven-day trial or asking a developer to fiddle the database date by hand, once per project, every time the logic changes.

### Solution

```gherkin
  Scenario: Trial banner flips after 7 days
    Given the system time is "2026-01-01T00:00:00Z"
    And the active trial started at "2025-12-25T00:00:00Z"
    And I am on "/dashboard"
    Then ".trial-banner" should contain text "Trial expires in 1 day"

  Scenario: Trial expires
    Given the system time is "2026-01-02T00:00:00Z"
    And the active trial started at "2025-12-25T00:00:00Z"
    And I am on "/dashboard"
    Then ".trial-banner" should contain text "Trial expired" within 2 seconds
```

Two scenarios, two different "today" values, one fixed trial-start fixture, same code path. Because the banner logic runs at load time, no advancing is needed -- each scenario installs its chosen instant *before* navigation, so the page's expiry calculation reads the mocked date on its very first `new Date()`. The pair brackets the boundary: one day before expiry and the day after. If you want to be thorough, add the exact-boundary scenario at `2026-01-01T00:00:00Z` -- off-by-one bugs live precisely there, in the difference between `>` and `>=`. As in Recipe 369, `the active trial started at ...` is your own fixture step seeding app state; the clock steps only control what the page thinks "now" is.

---

## Recipe 374: Avoiding Real Sleeps in Date Tests

You inherit a suite where someone solved every timer with a real wait. It worked when there were five tests. Now there are a hundred, the nightly run takes two hours, and most of that time is the suite doing literally nothing -- sitting out countdowns that a mocked clock would cross instantly.

### Anti-pattern

```gherkin
  Scenario: BAD -- waits 60 real seconds
    Given I am on "/verify"
    When I click "Send code"
    And I wait for 60 seconds
    Then "#resend" should be enabled
```

This test runs for 60 seconds of wall time. In a suite of 100 such tests, that's 100 minutes of CI burn for nothing.

### Better

```gherkin
  Scenario: GOOD -- advances mock clock
    Given the system time is "2026-01-01T00:00:00Z"
    And I am on "/verify"
    When I click "Send code"
    And I advance the clock by 60 seconds
    Then "#resend" should be enabled within 1 second
```

The lesson is simple: any time you reach for `I wait for N seconds`, ask whether `I advance the clock by N seconds` would do the same job in zero wall time. This is also the house rule of webship-js itself -- the framework ships with no static sleeps anywhere; every built-in wait goes through `smartSettle`, which watches AJAX counts, pending timers, and DOM mutations instead of the wall clock. A real 60-second wait is not even safer: it still races the thing it waits for, it just races slowly. The clock advance does not race anything -- it *causes* the event, then asserts.

**At scale:** static sleeps are the cheapest suite-wide cost to eliminate, and the easiest to enforce mechanically. Grep your feature files for `I wait for` in the CI pipeline and fail the lint when one appears outside an approved allowlist. The arithmetic sells itself to whoever owns the CI bill: a hundred 60-second sleeps is 100 minutes of farm time per run, per project, every night -- across 40 suites, that is a number with a currency sign on it, spent on waiting.

---

## Common Clock Patterns

| Goal | Pattern |
|------|---------|
| Freeze for a deterministic page snapshot | `Given the system time is "X"` |
| Fire scheduled callback now | `When I advance the clock by N ms` |
| Cross a calendar boundary | `When I advance the clock by N hours` |
| Fast-forward everything queued | `When I wait until pending timers settle` |
| Stop the in-page clock from drifting | `And I pause the clock` |
| Hand back to real time | `When I resume the clock` |

Use the clock mock anywhere your test logic depends on "what time is it now." Real time is for end-to-end smoke tests and human users. Test time is yours to control.

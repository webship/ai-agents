# Chapter 36: Web-First Assertions

---

In older test frameworks, the rhythm was always the same: wait for AJAX, then assert. If you skipped the wait, the assertion fired against stale DOM and your test failed. Webship-js ~2.0 ships a family of **web-first assertions** that fold the wait into the assertion itself. You write a single line. It polls the page every 100ms until the matcher passes or the time budget runs out -- 5 seconds by default, or whatever you set with `within N seconds`.

The assertion IS the wait. No more "wait for AJAX, then check." No more arbitrary `I wait 3 seconds`. And here is the part teams keep missing: the budget is a cap, not a sleep. A passing assertion returns on the first successful poll. A 10-second budget on a healthy page costs you about 100 milliseconds.

This chapter is built around the new web-first family added in the latest webship-js release: state checks (`visible`, `hidden`, `focused`, `enabled`, `disabled`, `editable`, `attached`), value/text/attribute/class/count assertions, role-aware visibility, and viewport-membership checks -- each with an optional `within N seconds` budget override.

---

## Why Web-First Assertions Matter

<!-- diagram: bdd-flow -->

A traditional Selenium-era test reads like a transcript of computer fights:

```
1. Click button
2. Wait 2 seconds (hope for the best)
3. Check if the success message is there
4. If not, sleep 2 more seconds and check again
```

A web-first test reads like a behaviour description:

```gherkin
When I click "Submit"
Then ".success-banner" should be visible within 10 seconds
```

Same intent. Half the steps. Zero flakiness from arbitrary sleeps.

There is a second layer working underneath, and knowing it explains why these assertions feel so calm. After every state-changing step, webship-js runs `smartSettle(page, 1500)` -- it watches `window.__webshipAjaxCount` (every fetch and XHR in flight), `window.__webshipPendingTimers` (queued `setTimeout` callbacks), and `window.__webshipLastMutation` (the timestamp of the last DOM change), and waits until the page has been quiet for at least 250ms. So by the time your `Then` runs, most of the storm has already passed. The web-first assertion covers the one thing settle cannot know: whether *your* element reached *your* expected state. Settle stabilises; the assertion verifies. You need both, and you get both without writing either.

---

## Recipe 343: Asserting Element Visibility With a Budget

### Problem
Your contact form posts to a backend that answers in 200ms on a quiet Tuesday morning and in 8 seconds during the load spike before a release. The test you wrote on your laptop passes every time; the same test fails twice a week in the nightly run, and every triage ends with "it's fine, just re-run it." The old fix was `I wait 10 seconds`, which made every run slow to protect against the worst case.

### Solution

```gherkin
Feature: Submit form

  Scenario: Successful submission shows banner
    Given I am on "/contact"
    When I fill in "name" with "Maria"
    And I fill in "email" with "maria@example.com"
    And I press "Send"
    Then ".success-banner" should be visible within 10 seconds
```

The poll loop checks visibility every 100ms until the banner appears or the 10-second deadline passes. Default budget without `within` is 5 seconds. The first successful poll wins immediately -- on a fast backend this step finishes in a few hundred milliseconds even with a 10-second budget. And most of the waiting has often already happened before the assertion runs: the AfterStep hook ran `smartSettle(page, 1500)` after the click, watching `window.__webshipAjaxCount` drop back to zero. The assertion covers what settle cannot -- the render that happens *after* the response lands. One ceiling to respect: the Cucumber step timeout is 45000ms, so keep `within` budgets comfortably under 45 seconds.

**At scale:** standardise budgets instead of letting each author guess. A sensible portfolio rule: default 5 seconds everywhere, plus a short, documented exception list for known-slow endpoints. Generous budgets cost nothing on green runs -- polling exits on first pass -- they only delay the discovery of true failures, which is the right trade. What hurts across 20 projects is twenty different opinions about what "slow" means, sprinkled as random `within` clauses nobody can review.

---

## Recipe 344: Asserting Element Is Hidden

### Problem
Your settings modal closes with a 300ms fade-out. The element stays in the DOM the whole time -- the framework keeps it mounted and toggles CSS. A test that checks for absence right after Escape fails randomly on slower CI machines because the animation has not finished, and passes on your laptop because it has. Classic "works on my machine," except this time the machine works and the test does not.

### Solution

```gherkin
  Scenario: Modal closes on Escape
    Given I am on "/dashboard"
    When I click "Open Settings"
    And I press the key "Escape"
    Then ".modal" should be hidden within 2 seconds
```

`hidden` accepts both "removed from DOM" and "present but not visible" -- mirroring what a real user perceives. The matcher polls every 100ms, so the 300ms fade costs three polls and the step passes the moment `display: none` (or detachment) lands. This is also why you assert `hidden` rather than checking the element left the DOM: modern frameworks keep modals mounted for reuse, and a DOM-absence check would fail forever against a perfectly working close button.

---

## Recipe 345: Asserting Element Is Attached

### Problem
Your product table renders 2,000 rows through a virtualised list -- DOM nodes mount only as the user scrolls near them. The new hire writes `should be visible` for row 50, watches it fail, and spends an hour convinced the data is missing. The data is fine. The row mounts after scrolling but sits outside the viewport, and "visible" means exactly what it says.

### Solution

```gherkin
  Scenario: Row 50 mounts after scroll
    Given I am on "/products"
    When I scroll to bottom
    Then "[data-row='50']" should be attached within 5 seconds
```

`attached` checks DOM presence and nothing else -- no CSS visibility, no viewport math. The poll asks "does a node matching this selector exist in the document?" every 100ms. That is exactly the right question for virtualised lists, where mounting is the behaviour under test. Use `visible` when you care about what the user actually sees, `attached` when you care about what the framework actually rendered. Mixing them up is the most common reason a correct page fails a correct-looking test.

---

## Recipe 346: Asserting Focused Element

### Problem
After form validation fails, the first invalid field should receive focus so a keyboard user lands exactly where the fix is needed. This contract breaks silently: a refactor swaps the validation library, focus stops moving, and no sighted mouse user ever notices. The bug report arrives weeks later from a screen-reader user -- or from the accessibility audit you are paying for.

### Solution

```gherkin
  Scenario: Invalid email re-focuses email field
    Given I am on "/signup"
    When I fill in "email" with "not-an-email"
    And I press "Sign up"
    Then "#email" should be focused within 1 second
```

Focus is a keyboard-accessibility contract. The web-first matcher polls `document.activeElement` every 100ms until it matches your selector, so the assertion tolerates the small delay between the validation handler running and the `.focus()` call landing. The tight 1-second budget is deliberate -- focus management is near-synchronous, and a long budget would only slow the discovery of a real regression.

**At scale:** make a focus assertion part of the definition of done for every form across your products. It is one line per form, it costs nothing on green runs, and it catches the class of regression that otherwise reaches you through support tickets from your least-served users. Pair it with the audit step (`Then the page should pass an accessibility audit at level "AA"`) -- the audit catches structural issues, the focus assertion catches behavioural ones.

---

## Recipe 347: Asserting Enabled / Disabled

### Problem
A "Save" button stays disabled until the form is valid. Two sprints ago someone refactored the validation logic, and on one of your sites the button never enabled again -- users filled in everything and stared at a gray button. It took three days and one angry customer to surface, because every manual tester filled the form in the one "right" order that happened to still work.

### Solution

```gherkin
  Scenario: Save unlocks after required fields
    Given I am on "/profile"
    Then "#save" should be disabled
    When I fill in "name" with "Lina"
    And I fill in "email" with "lina@example.com"
    Then "#save" should be enabled within 2 seconds
```

`disabled` and `enabled` are mirror predicates sharing the same 100ms polling logic. They read the element's real disabled state -- the `disabled` property as the browser sees it -- not the styling. That distinction matters both ways: a button styled gray but still clickable is a bug this catches, and a button styled normally but dead to clicks is the other bug it catches. Note the first assertion carries no `within` clause: it uses the default 5-second budget and passes on the first poll, because the page arrived settled -- navigation already ran through smartSettle before the step executed.

---

## Recipe 348: Asserting Editable

### Problem
Profile fields ship read-only until the user clicks "Edit". After a design-system upgrade, the field *looked* editable -- live border, text cursor, the works -- but the `readonly` attribute never came off. Users clicked, typed into a void, and assumed they had saved. An "enabled" assertion would have passed the whole time; the field was enabled. You need the stronger check.

### Solution

```gherkin
  Scenario: Edit unlocks the input
    Given I am on "/profile"
    Then "input[name=display-name]" should not be editable
    When I click "Edit"
    Then "input[name=display-name]" should be editable within 2 seconds
```

`editable` is stronger than `enabled` -- it checks the `readonly` attribute, the `disabled` attribute, and the contentEditable state in one call, polled on the same 100ms cadence. An input can be enabled yet read-only, and a rich-text `div` can be neither an input nor disabled yet still refuse typing. `editable` answers the only question the user cares about: if I type, do characters appear? The negative form before the click pins the starting state, so the test proves the *transition* -- not just wherever the field happened to end up.

---

## Recipe 349: Text-Containment Assertion With Budget

### Problem
After clicking "Apply", a coupon summary updates somewhere in the cart sidebar. Marketing rewrites the surrounding copy every quarter -- "You saved", "Discount applied", "Your savings" -- and every rewrite used to break the old exact-match test. The one thing that never changes is the discount amount itself. Anchor on that.

### Solution

```gherkin
  Scenario: Coupon discount appears
    Given I am on "/cart"
    When I fill in "code" with "SAVE10"
    And I click "Apply"
    Then ".cart-summary" should contain text "Discount: -$10.00" within 5 seconds
```

The matcher does a substring match against the element's rendered text, re-read on every 100ms poll -- so debounced updates, streamed totals, and slow re-renders are all caught the moment the substring appears. Scope the selector to the tightest stable container you have: a substring asserted against `body` will eventually pass for the wrong reason -- an error message carrying the same digits, a cached value elsewhere on the page. For regex or whole-text equality, use the dedicated steps in the next two recipes.

**At scale:** substring assertions and translated sites do not mix. "Discount" is "Rabatt" on your German brand site and something else entirely on the Arabic one. When the same scenario must run across locales, anchor on locale-stable substrings -- amounts, currency codes, SKUs -- or resolve elements through your shared selectors JSON in `tests/selectors/`, and keep the human words out of the assertion.

---

## Recipe 350: Exact Text Assertion

### Problem
A status badge should read exactly `Active`. The old test used a substring check, and `Inactive` contains `Active` -- so when a deploy inverted the status logic, the suite stayed green while every dashboard in production showed the wrong state. One character of slack, one afternoon of incident review.

### Solution

```gherkin
  Scenario: Status badge is exact
    Given I am on "/users/42"
    Then ".status-badge" should have text "Active" within 2 seconds
```

`should have text` does exact, whitespace-trimmed equality on every poll. The trim matters more than it looks: templates pad text nodes with newlines and indentation, and without trimming, `"\n  Active\n"` would never equal `"Active"`. Use `should contain text` when you genuinely want substring semantics -- and reach for exact equality any time one valid value is a substring of another, which happens constantly with status words (Active/Inactive, Done/Undone, Lock/Unlock).

**At scale:** status labels are a vocabulary, and vocabularies drift. One product says "Active", the next says "Enabled", the third says "On". Keep a short glossary of canonical status strings per product family and let exact-text assertions enforce it on every run -- when a developer "improves" a label, the suite tells you before the translations, the documentation, and the screenshots all start disagreeing with each other.

---

## Recipe 351: Value Assertion on Inputs

### Problem
A URL slug field auto-populates from the title field as the user types. The transformation runs on a 300ms debounce.

### Solution

```gherkin
  Scenario: Slug auto-populates from title
    Given I am on "/articles/new"
    When I fill in "title" with "Hello World"
    Then "input[name=slug]" should have value "hello-world" within 1 second
```

For form inputs, `should have value` reads the live `.value` property (not the `value=` attribute), so it sees user input and JavaScript writes.

---

## Recipe 352: Class Assertion With Budget

### Problem
After clicking "Save", a row gets the class `.row--saved` for a brief flash before the toast appears. Catching the transient class proves the visual feedback ran.

### Solution

```gherkin
  Scenario: Row flashes saved class
    Given I am on "/inventory"
    When I click "Save" in the "SKU-12" row
    Then "tr[data-sku='SKU-12']" should have class "row--saved" within 1 second
```

The poll checks `element.classList.contains('row--saved')` every 100ms.

---

## Recipe 353: Attribute Assertion

### Problem
An `<img>` lazy-loads. Its `src` attribute starts as a 1x1 placeholder and becomes the real image URL once the image enters the viewport.

### Solution

```gherkin
  Scenario: Lazy image swaps src
    Given I am on "/gallery"
    When I scroll to "img[data-id='hero']"
    Then "img[data-id='hero']" should have attribute "src" with value "https://cdn.example.com/hero.jpg" within 5 seconds
```

For exact attribute matching. The check uses `element.getAttribute('src')`, so it sees both initial and JavaScript-updated values.

---

## Recipe 354: Element Count Assertion

### Problem
You filter a list down to all rows starting with "A". You want to assert the list has exactly 7 results.

### Solution

```gherkin
  Scenario: Filter narrows to seven rows
    Given I am on "/customers"
    When I fill in "filter" with "A"
    Then ".customer-row" should have a count of 7 within 3 seconds
```

The matcher polls `await page.locator('.customer-row').count()` until it equals 7 or the budget elapses.

---

## Tip Round-up

| Need | Step |
|------|------|
| Element on screen | `"sel" should be visible` |
| Element gone | `"sel" should be hidden` |
| Element in DOM, any visibility | `"sel" should be attached` |
| Keyboard focus | `"sel" should be focused` |
| Click target | `"sel" should be enabled` / `disabled` |
| Type-able input | `"sel" should be editable` |
| Substring text | `"sel" should contain text "X"` |
| Exact text | `"sel" should have text "X"` |
| Input value | `"sel" should have value "X"` |
| CSS class | `"sel" should have class "X"` |
| Attribute value | `"sel" should have attribute "X" with value "Y"` |
| Count | `"sel" should have a count of N` |
| Viewport membership | `"sel" should be in the viewport` |
| Role visibility | `the "OK" button should be visible` |

Every matcher accepts a trailing `within N seconds` clause. Default is 5 seconds. Use a larger budget for slow back-ends; never zero.

---

Web-first assertions retire two old habits: `wait for AJAX` before checks, and `I wait 3 seconds` filler. Both still exist for legacy code, but new tests should default to the matchers in this chapter.

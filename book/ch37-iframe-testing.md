# Chapter 37: Iframe Testing

---

Iframes are the cockroaches of web testing. Hard to kill, hard to ignore, and hidden in places you least expect: payment widgets, embedded video players, ad units, third-party comment systems, OAuth consent pages. The DOM inside an iframe is a separate document. Standard selectors do not cross that boundary. Click an iframe's button with a normal step, and Playwright reports "element not found" -- because as far as the parent document is concerned, the button doesn't exist.

Webship-js ~2.0 ships first-class iframe support. You switch into a frame, interact normally, then switch back to the root document. All steps inside an iframe behave exactly as they do on a top-level page. The frame context is per-scenario -- the next scenario starts fresh on the root document.

---

## How Frame Switching Works

<!-- diagram: page-lifecycle -->

Every interaction step in webship-js targets a "current frame." By default, that is the top-level document. When you switch to an iframe, the current frame becomes that iframe's content. All subsequent clicks, fills, and assertions resolve against the iframe's DOM. When you switch back to the root document, you return to the top-level page.

```
┌──────────────────────────────────┐
│  Root Document                   │
│                                  │
│  <h1>Checkout</h1>               │
│                                  │
│  ┌─────────────────────────┐     │
│  │ <iframe name="payment"> │     │
│  │   <form>                │     │
│  │     <input name="card"> │     │
│  │     <button>Pay</button>│     │
│  │   </form>               │     │
│  └─────────────────────────┘     │
│                                  │
└──────────────────────────────────┘
```

Without switching, `When I fill in "card" with "4242..."` fails. After `switch to the iframe with name "payment"`, the same step succeeds.

Under the hood, the switch steps are built on Playwright's `frameLocator`. When you switch to an iframe, webship-js stores a frame-scoped locator for the scenario -- effectively `page.frameLocator('iframe[name="payment"]')` -- and every later step resolves its selectors through that scope instead of the page. Two properties of `frameLocator` matter in practice. First, it is lazy: the iframe element is re-resolved on every action, so a widget that reloads itself mid-scenario does not leave you holding a stale frame reference. Second, it is strict about existence: if the iframe never attaches, the step fails with a clear "frame not found" instead of silently clicking the wrong document. Switching to the root document simply drops the stored scope, and the per-scenario reset means the next scenario always starts on the top-level page -- no leakage between tests.

---

## Recipe 355: Switching to an Iframe by Name

### Problem
The Tuesday before release, your checkout suite goes red on every project that uses the new payment provider. The error says `cardNumber` cannot be found, and a teammate burns an hour "fixing" selectors that were never broken. The real cause: the provider ships its card input as `<iframe name="card-frame">`, and your fill steps are searching the parent document for a field that lives in a different one.

### Solution

```gherkin
Feature: Checkout via payment iframe

  Scenario: Pay with card
    Given I am on "/checkout"
    When I switch to the iframe with name "card-frame"
    And I fill in "cardNumber" with "4242424242424242"
    And I fill in "expiry" with "12/30"
    And I fill in "cvc" with "123"
    And I switch to the root document
    And I click "Place Order"
    Then ".order-confirmation" should be visible within 10 seconds
```

The switch step builds `page.frameLocator('iframe[name="card-frame"]')` and stores it as the scenario's current frame. From that point, `I fill in "cardNumber"` runs the same label-first field resolution it always does -- label, then id, name, placeholder -- just scoped to the iframe's DOM instead of the page's. Auto-waiting still applies inside the frame: the fill waits for the field to be attached, visible, and editable before typing, and the AfterStep `smartSettle` still watches the parent page's AJAX counter and mutation timestamp, so a parent-side reaction to the iframe (a price recalculation, say) settles before the next step.

The `switch to the root document` step is critical. If you forget it, subsequent steps still target the iframe and will not find the "Place Order" button.

**At scale:** payment iframes are the most duplicated test code in a multi-project portfolio -- every commerce site you run has the same switch-fill-fill-fill-switch dance. Put the iframe's selector in a shared JSON file under `tests/selectors/` (loaded through the `worldParameters.selectors` registry) and, if the provider is the same everywhere, wrap the whole dance in one custom step. When the provider renames its frame, you fix one file and twenty checkouts go green again.

---

## Recipe 356: Switching to an Iframe by Title

### Problem
The third project you onboard uses the same payment vendor as the first two -- but a newer embed version, and this one ships its iframe with no `name` attribute at all. The only thing it carries is `title="Payment form"`, added for screen readers. Your by-name step from the last recipe has nothing to grab.

### Solution

```gherkin
  Scenario: Pay via iframe with no name attribute
    Given I am on "/checkout"
    When I switch to the iframe with title "Payment form"
    And I fill in "cardNumber" with "4242424242424242"
    And I switch to the root document
```

The step resolves to `page.frameLocator('iframe[title="Payment form"]')` -- same scoping mechanics as the by-name variant, different attribute. `title` is the second most reliable iframe identifier after `name`, and for the same reason your accessibility audits care about it: an iframe's `title` is what a screen reader announces, so changing it is an a11y regression, not a refactor. Vendors and designers leave it alone. If your suite already runs `the page should pass an accessibility audit at level "AA"`, you get a free early warning -- the audit flags a missing iframe title before your switch step ever fails on it.

---

## Recipe 357: Switching by Locator

### Problem
Then there is the iframe with neither `name` nor `title` -- usually one your own front-end team generated, because nobody writes those attributes unless a linter makes them. The new hire's first PR targets it with `.css-x7k2j > iframe:nth-child(2)`, which works until the next style refactor. The only identifier that will survive a redesign is a CSS selector on something deliberate, like `iframe[data-component="stripe-card"]`.

### Solution

```gherkin
  Scenario: Switch by CSS selector
    Given I am on "/checkout"
    When I switch to iframe with locator "iframe[data-component='stripe-card']"
    And I fill in "cardNumber" with "4242424242424242"
    And I switch to the root document
```

The locator string goes straight into `page.frameLocator(...)`, so anything Playwright accepts as a selector works here -- CSS, attribute selectors, the lot. That makes this the escape hatch when `name` and `title` are missing, and the foot-gun when you feed it a generated class name. Prefer `data-*` attributes: they exist only for hooks like this, so no designer has a reason to rename them.

**At scale:** raw CSS strings inside feature files are where multi-project maintenance goes to die. Register the locator once in a shared selectors JSON file (`{"css": {"payment_frame": "iframe[data-component='stripe-card']"}}` in `tests/selectors/`, auto-loaded via `worldParameters.selectors.files`) and reference the name. When the vendor changes its embed markup, the diff that fixes all your projects is one line in one file -- and the review takes thirty seconds instead of a selector hunt across twenty repositories.

---

## Recipe 358: Clicking a Button Inside an Iframe

### Problem
An order flow ends in a confirmation iframe: click "Confirm" inside it, then check the "Order accepted" message that renders in the same frame. Doing this with bare steps works, but six months later the scenario reads like every other click-and-check, and the reviewer approving a change cannot tell which steps run inside the frame and which on the page. You want the click, the assertion, and the frame boundary to be visible in the Gherkin itself.

### Solution

```gherkin
  Scenario: Confirm inside iframe
    Given I am on "/orders/new"
    When I switch to the iframe "confirmation"
    And I click "Confirm" inside the iframe
    Then I should see "Order accepted" inside the iframe
    And I switch to the root document
```

The `inside the iframe` suffix tells the step definition to resolve through the scenario's current `frameLocator` rather than the page -- and it tells the human the same thing. Mechanically, the click is a normal Playwright click with full actionability checks: the button must be attached, visible, stable, and enabled inside the frame before the click fires, with the usual web-first polling while it gets there. The suffix costs nothing at runtime; what it buys you is a feature file where the frame boundary is explicit, which is exactly the line a product owner or a reviewer needs when they ask "where does the third-party part start?"

---

## Recipe 359: Asserting Text Inside an Iframe

### Problem
Marketing swaps the live-chat vendor on a Friday. The new widget loads, the page looks fine, and nobody notices for two weeks that the chat bubble says nothing because the embed token was wrong -- until a customer mentions it. A one-line smoke check that the widget's iframe actually renders "We're online" would have caught it on the first nightly run.

### Solution

```gherkin
  Scenario: Live chat shows online status
    Given I am on "/"
    When I switch to the iframe with title "Support chat"
    Then I should see "We're online" inside the iframe
    And I switch to the root document
```

The `inside the iframe` variant of `I should see` searches the iframe's body text through the stored `frameLocator`, not the parent page. Third-party widgets boot slowly -- they load their own scripts, open their own connections -- so the web-first polling matters here: the assertion retries every 100ms (up to the default 5 seconds) while the widget initializes, instead of failing on the first empty paint. No sleep, no flake.

**At scale:** the same chat widget, cookie banner, or survey embed usually appears on most of the sites you maintain. Keep these checks in one widget-smoke feature, tag the scenarios with something like `@third-party`, and reuse the file across projects with `LAUNCH_URL` pointing at each site. The tag earns its keep in CI: exclude `@third-party` from PR pipelines (a vendor outage should not block your developers' merges) and run it nightly, where a failure means an email to the vendor instead of a reverted release.

---

## Recipe 360: Filling a Field Inside an Iframe by Attribute

### Problem
An embedded comment widget renders a textarea with no label, no placeholder, no id -- just `data-role="comment-body"`, because the vendor's accessibility story is "we'll get to it". Label-first resolution has nothing to latch onto, and you don't control the widget's markup, so fixing the HTML is not on the table. You need to fill and submit through the attribute you actually have.

### Solution

```gherkin
  Scenario: Submit a comment via embedded widget
    Given I am on "/articles/42"
    When I switch to the iframe "comments"
    And I fill in "comment-body" with "Loved the article" inside the iframe
    And I click "Post" inside the iframe
    Then I should see "Comment posted" inside the iframe
    And I switch to the root document
```

The combined `inside the iframe` step lets you target attribute-based fields without breaking out of the frame. Resolution still walks the usual chain -- label, id, name, placeholder, then attribute values -- it just walks it inside the iframe's document, so `comment-body` matches the `data-role` value even though no label exists. Everything else stays scenario-normal: the fill waits for editability, the click on "Post" waits for actionability, and the closing assertion polls for the success message inside the same frame. One scenario, one behaviour, frame boundary explicit at both ends.

---

## Recipe 361: Clicking by Attribute Inside an Iframe

### Problem
A sponsored iframe drops over your page's hero section, and until it is dismissed it intercepts clicks meant for the content underneath -- so half your homepage scenarios fail with "element is covered" on days when the ad server is feeling generous. The close button inside the ad has no text, only an icon and `aria-label="Close ad"`. You need to dismiss it by that attribute, then prove the overlay is gone.

### Solution

```gherkin
  Scenario: Dismiss ad iframe
    Given I am on "/"
    When I switch to the iframe with title "Sponsored"
    And I click "Close ad" by attr inside the iframe
    Then "iframe[title='Sponsored']" should be hidden within 3 seconds
```

The `by attr` variant matches the click target against attribute values -- `aria-label`, `title`, `data-*` -- instead of visible text, which is exactly what icon-only buttons need. The closing assertion uses `should be hidden`, the web-first matcher that passes when the element is invisible *or* detached entirely; an iframe that removes itself from the DOM satisfies it either way, polled every 100ms until it does. One quirk to know: after clicking inside an iframe that removes itself, you would normally still need `switch to the root document` -- but if the current frame is gone, the next step falls back to the root automatically rather than failing on a dead frame reference.

**At scale:** before you copy this recipe into twenty suites, ask whether the ad needs to exist in your tests at all. If the ad's behaviour is not what you are testing, the cheaper fix is one shared `Background` line that blocks the ad network's URL with the network-mock step from Chapter 39 -- no iframe ever loads, no overlay ever intercepts a click, and every page loads faster in CI. Reserve the click-to-dismiss pattern for the one suite where the ad experience itself is under test.

---

## Recipe 362: Negative Iframe Text Assertion

### Problem
A returning customer navigates back to the checkout page and the payment iframe greets them with the previous card number, pre-filled from a caching layer that should never have touched it. That is not a UI bug; that is the kind of incident that gets its own postmortem and a very uncomfortable compliance call. You want a standing smoke test that proves card digits never echo back into the payment frame after navigation.

### Solution

```gherkin
  Scenario: Card field empty after navigation
    Given I am on "/checkout"
    When I switch to the iframe "payment"
    Then I should not see "4242" inside the iframe
    And I switch to the root document
```

`should not see ... inside the iframe` is the negation matcher, scoped to the current frame's body text. It returns immediately when the text is absent and fails the moment it is found -- which makes it nearly free to run, so there is no runtime excuse to skip it. One honest caveat: a negative text assertion proves absence *at the moment it runs*. Place it after the navigation that could trigger the leak, not before, or it passes without testing anything.

**At scale:** checks like this are evidence, not just tests. Tag them `@security` or `@compliance`, run the tag across every project nightly, and stamp the run with report metadata (`--metadata` flags on the report generator) so the cross-project dashboard carries a per-product "no sensitive data echoed" line. When an auditor -- or your PMO -- asks how you know card data never leaks into the page, "a tagged scenario asserts it on every site, every night" is an answer; "we reviewed the code in March" is not.

---

## Recipe 363: Nested Iframes

### Problem
A pricing widget arrives as an iframe, and somewhere inside it the vendor embedded *another* iframe for the live-chat trigger. Iframes in iframes -- the test that needs to click "Talk to sales" has to cross two document boundaries, and the first time someone tries it with a single switch, the step fails and the debugging session runs long because the button is visible right there on the screen.

### Solution
Webship-js currently supports one level of iframe switching at a time. To reach a nested iframe, switch to the outer first, then re-target the inner -- the resolver matches against frames at every depth.

```gherkin
  Scenario: Click chat trigger inside nested iframe
    Given I am on "/pricing"
    When I switch to the iframe "pricing-widget"
    And I switch to the iframe "chat-trigger"
    And I click "Talk to sales" inside the iframe
    And I switch to the root document
```

Each switch re-scopes from wherever you currently stand. The first switch stores a `frameLocator` for the pricing widget; the second resolves "chat-trigger" from inside that scope, chaining frame locators the way Playwright chains `frameLocator().frameLocator()`. Both hops stay lazy -- each level re-resolves on every action -- so a chat widget that re-renders its inner frame does not stale the chain. The single `switch to the root document` at the end unwinds the whole stack; you do not exit one level at a time.

> **Note:** This pattern works because the iframe resolver searches all frames at every depth. For deeply nested chains, design your fixtures so the inner iframe has a unique name -- a one-hop switch to a uniquely named frame is far less brittle than a multi-hop chain that breaks whenever the vendor restructures its embed.

---

## Recipe 364: Returning to the Root Document

### Problem
A scenario fails on `I click "Place Order"` with "element not found", and the button is plainly there when you open the page yourself. Someone re-runs it, someone blames the environment, someone adds a longer timeout that changes nothing. The actual bug is three lines up: the scenario switched into the payment iframe and never switched back, so the click is hunting for "Place Order" inside a card widget that has no such button.

### Solution

```gherkin
  Scenario: Iframe round trip
    Given I am on "/checkout"
    When I switch to the iframe "payment"
    And I fill in "cardNumber" with "4242424242424242"
    And I switch to the root document
    And I click "Place Order"
    Then ".confirmation" should be visible within 10 seconds
```

`switch to the root document` drops the stored `frameLocator` and points the scenario back at the page object itself -- the cheapest step in the suite, and the one whose absence costs the most afternoons. Forgetting it is the single most common iframe-test bug, and the symptom is always the same: "element not found" on a step that targets the parent page, right after an iframe block. The frame scope is per-scenario, so the damage never leaks into the next scenario -- which is exactly why the bug is confusing: scenario B passes while scenario A fails on a "missing" element that both of them use.

**At scale:** make the pairing a review convention -- every `switch to the iframe` must have a matching `switch to the root document` in the same scenario, the way every opening brace has a closing one. This is mechanical enough to automate: a few lines of script over your `.feature` files can count switches per scenario and fail the pipeline on an unbalanced pair. Cheap lint, and it deletes an entire class of "works on my machine" tickets across every project that touches an iframe.

---

## Iframe Cheat Sheet

| Step | Use case |
|------|----------|
| `I switch to the iframe "name"` | iframe has a `name` attribute |
| `I switch to the iframe with title "X"` | iframe has only a `title` |
| `I switch to iframe with locator "sel"` | iframe identified by CSS |
| `I switch to the root document` | exit iframe context |
| `I click "X" inside the iframe` | click visible label |
| `I click "X" by attr inside the iframe` | click by attribute |
| `I fill in "X" with "Y" inside the iframe` | fill field |
| `I should see "X" inside the iframe` | text present |
| `I should not see "X" inside the iframe` | text absent |

End every iframe block with `switch to the root document`. Treat it like closing a brace -- non-negotiable.

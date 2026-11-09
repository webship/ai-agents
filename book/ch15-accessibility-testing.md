# Chapter 15: Accessibility Testing

---

Accessibility is not a feature. It is a fundamental quality of software. When your login form cannot be navigated with a keyboard, you have excluded every user who relies on one. When your images lack alt text, screen readers announce nothing. When your color contrast is insufficient, users with low vision cannot read your content.

Automated accessibility testing catches the most common issues before they reach production. This chapter shows you how to integrate accessibility checks into your webship-js test suite.

---

## Understanding Web Accessibility

```
┌──────────────────────────────────────────────────────────────────┐
│ The Four Principles of Web Accessibility (POUR)                  │
│ From WCAG 2.1                                                    │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ P - Perceivable  │  │ O - Operable     │                    │
│  │                  │  │                  │                    │
│  │ Users must be    │  │ Users must be    │                    │
│  │ able to perceive │  │ able to operate  │                    │
│  │ the content      │  │ the interface    │                    │
│  │                  │  │                  │                    │
│  │ • Alt text       │  │ • Keyboard nav   │                    │
│  │ • Captions       │  │ • No time limits │                    │
│  │ • Color contrast │  │ • No seizure risk│                    │
│  └──────────────────┘  └──────────────────┘                    │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ U - Understandable│  │ R - Robust       │                    │
│  │                  │  │                  │                    │
│  │ Content and UI   │  │ Content must     │                    │
│  │ must be          │  │ work with        │                    │
│  │ understandable   │  │ current and      │                    │
│  │                  │  │ future tech      │                    │
│  │ • Clear language │  │ • Valid HTML     │                    │
│  │ • Predictable    │  │ • ARIA roles     │                    │
│  │ • Error guidance │  │ • Standard APIs  │                    │
│  └──────────────────┘  └──────────────────┘                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Recipe 154: Testing Keyboard Navigation

**Problem:** A user who cannot use a mouse Tabs into your login form and gets stuck -- focus jumps from Email straight into a hidden newsletter field, then vanishes into an off-screen carousel. No functional test notices, because functional tests click. The first time anyone discovers it is an accessibility audit, a procurement questionnaire, or a complaint -- and every one of those costs more than the test would have.

**Solution:**

```javascript
// tests/step-definitions/custom-keyboard.js
const { When, Then } = require('@cucumber/cucumber');
const assert = require('assert');

When('I press the Tab key', async function () {
  await this.page.keyboard.press('Tab');
});

When('I press the Tab key {int} times', async function (count) {
  for (let i = 0; i < count; i++) {
    await this.page.keyboard.press('Tab');
  }
});

When('I press the Enter key', async function () {
  await this.page.keyboard.press('Enter');
});

When('I press the Escape key', async function () {
  await this.page.keyboard.press('Escape');
});

Then('the focused element should be {string}', async function (expectedText) {
  const focused = await this.page.evaluate(() => {
    const el = document.activeElement;
    return el ? (el.textContent || el.value || el.placeholder || el.getAttribute('aria-label') || '').trim() : '';
  });
  assert(focused.includes(expectedText),
    `Expected focused element to contain "${expectedText}", but got "${focused}"`);
});

Then('the focused element should have role {string}', async function (expectedRole) {
  const role = await this.page.evaluate(() => {
    const el = document.activeElement;
    return el ? (el.getAttribute('role') || el.tagName.toLowerCase()) : '';
  });
  assert.strictEqual(role, expectedRole,
    `Expected focused element role "${expectedRole}", but got "${role}"`);
});
```

```gherkin
Feature: Keyboard Navigation
  As a keyboard-only user
  I want to navigate the site without a mouse
  So that I can use all features

  Scenario: Tab through the login form
    Given I am on homepage
    When I go to "/login"
    And I press the Tab key
    Then the focused element should be "Email"
    When I press the Tab key
    Then the focused element should be "Password"
    When I press the Tab key
    Then the focused element should be "Sign In"

  Scenario: Submit form with Enter key
    Given I am on homepage
    When I go to "/login"
    And I fill in "Email" with "admin@example.com"
    And I fill in "Password" with "AdminPass123"
    And I press the Enter key
    And wait for AJAX to finish
    Then I should see "Dashboard"

  Scenario: Close modal with Escape key
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And I press "Add User"
    And wait for the modal to appear
    Then I should see a modal dialog
    When I press the Escape key
    And wait for the modal to disappear
    Then I should not see a modal dialog

  Scenario: Navigate menu with keyboard
    Given I am on homepage
    When I press the Tab key 3 times
    And I press the Enter key
    And wait for AJAX to finish
    Then I should see "Products"
```

**How It Works:**

`page.keyboard.press('Tab')` dispatches a real, trusted key event through the browser's input pipeline -- the browser itself decides where focus lands, following DOM order, tabindex, and visibility exactly as it would for a real user. That is the point: you are testing the browser's actual focus order, not your assumption of it. The assertion step then reads `document.activeElement` inside the page and checks its text, value, placeholder, or aria-label -- the same accessible-name sources a screen reader consults. The fallback chain in that step is deliberate: a focused element that has none of those is itself an accessibility bug, and the empty-string failure message will tell you so.

The Enter-key scenario covers a behavior teams break constantly: implicit form submission. Wire the click handler to the button instead of the submit event to the form, and Enter-to-submit dies -- only a keyboard test catches it. And because webship-js's AfterStep smartSettle runs after each key press, focus assertions land after any focus-shifting JavaScript has finished, not mid-transition.

**At scale:** Keyboard steps like these are exactly the custom code that gets copy-pasted into every project and then drifts. Keep them in one shared step-definition package, tag the scenarios @a11y, and run that tag everywhere. When a procurement questionnaire asks "do you test keyboard accessibility?", pointing at the same green tag across 20 products is a better answer than any policy document your PMO can draft.

---

## Recipe 155: Testing Focus Management

**Problem:** A modal opens and focus stays on the button behind it -- so a keyboard user's next Tab walks the page underneath a dialog they cannot see past. Or a form fails validation and the error message renders 800 pixels above the focused submit button, announced to no one. Focus management bugs are invisible in a mouse-driven sprint demo and crippling for everyone who navigates by keyboard.

**Solution:**

```gherkin
Feature: Focus Management
  Scenario: Focus moves to modal when opened
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And I press "Add User"
    And wait for the modal to appear
    # Focus should be on the first input in the modal
    Then the focused element should be "First Name"

  Scenario: Focus moves to error on validation failure
    Given I am on homepage
    When I go to "/login"
    And I press "Sign In"
    # Focus should jump to the first error
    Then I should see "Email is required"

  Scenario: Focus returns after modal closes
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And I press "Add User"
    And wait for the modal to appear
    When I close the modal
    And wait for the modal to disappear
    # Focus should return to the button that opened the modal
    Then the focused element should be "Add User"
```

**How It Works:**

These scenarios reuse the focused-element step from Recipe 154 to pin down the three moments WCAG cares about most: focus moves into a dialog when it opens, focus lands somewhere useful when an error appears, and focus returns to the trigger when the dialog closes. The third is the one most commonly broken -- close a modal and focus silently resets to `<body>`, stranding a keyboard user back at the top of the page with no idea where they were. webship-js's auto-settle earns its keep here: the AfterStep smartSettle waits for the modal's open and close animations (visible to it through window.__webshipPendingTimers and the DOM MutationObserver) before the next step runs, so you assert focus on the final state, never mid-transition.

---

## Recipe 156: Testing ARIA Labels and Roles

**Problem:** The page looks perfect and reads as garbage. A screen reader announces your icon-only search button as "button", your hero image as its filename, and offers no landmarks to jump between -- so navigating means listening to every element, in order, every visit. None of this shows up visually, which is exactly why it survives review after review until someone who depends on it files the bug.

**Solution:**

```javascript
// tests/step-definitions/custom-aria.js
const { Then } = require('@cucumber/cucumber');
const assert = require('assert');

Then('the {string} element should have aria-label {string}', async function (selector, label) {
  const ariaLabel = await this.page.locator(selector).first().getAttribute('aria-label');
  assert.strictEqual(ariaLabel, label,
    `Expected aria-label "${label}", got "${ariaLabel}"`);
});

Then('the page should have a {string} landmark', async function (role) {
  const count = await this.page.locator(`[role="${role}"], ${role}`).count();
  assert(count > 0, `No element with role="${role}" found on the page`);
});

Then('all images should have alt text', async function () {
  const images = await this.page.locator('img').all();
  for (const img of images) {
    const alt = await img.getAttribute('alt');
    const src = await img.getAttribute('src');
    assert(alt !== null && alt !== undefined,
      `Image "${src}" is missing alt attribute`);
  }
});
```

```gherkin
Feature: ARIA Compliance
  Scenario: Page has proper landmarks
    Given I am on homepage
    Then the page should have a "navigation" landmark
    And the page should have a "main" landmark

  Scenario: All images have alt text
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    And wait for the page to load
    Then all images should have alt text

  Scenario: Interactive elements are labeled
    Given I am on homepage
    When I go to "/login"
    Then the "#search-icon" element should have aria-label "Search"
```

**How It Works:**

The landmark step accepts both spellings of the same semantics -- explicit `role="navigation"` attributes and native elements like `<nav>` that carry the role implicitly -- which is why the locator queries `[role="${role}"], ${role}`. The alt-text step is stricter than it looks: it asserts the attribute *exists*, not that it contains text, because `alt=""` is the correct way to mark a decorative image, while a missing attribute makes screen readers fall back to announcing the filename. Before writing more checks like these, look at what ships in the box: webship-js 2.0 includes structural accessibility steps for landmarks, heading hierarchy, form labels, skip links, tabindex order, and ARIA references. Custom steps should encode your project's specific conventions, not re-implement the basics.

**At scale:** ARIA conventions are a team agreement, and agreements need enforcement to survive the third project you onboard. Put the landmark and labeling scenarios in a shared feature file every new project starts from -- they are application-agnostic by design -- and they become the cheapest accessibility regression net you will ever run: no test data, no login, just structure. New project, day one, already covered.

---

## Recipe 157: Testing Color Contrast

**Problem:** A brand refresh lightens the gray used for body text, and suddenly anyone with low vision -- or anyone reading a phone in sunlight -- cannot make out your content. WCAG AA requires 4.5:1 contrast for normal text and 3:1 for large text, and no human reviewer can eyeball a ratio. Contrast is the most common accessibility failure on the web precisely because it never looks "broken" to the people who built it.

**Solution:**

webship-js has a built-in CSS assertion step that can verify computed styles:

```gherkin
Feature: Color Contrast
  Scenario: Text is readable against background
    Given I am on homepage
    # Verify element has specific CSS properties
    Then the element "#main-heading" should have CSS "color" of "rgb(33, 37, 41)"
    And the element "#main-heading" should have CSS "font-size" of "32px"
```

For automated contrast ratio checking you do not need to build anything: webship-js 2.0 bundles axe-core and ships the audit as built-in steps -- `Then the page should pass an accessibility audit at level "AA"`, `Then the page should have no critical accessibility violations`, and `Then print accessibility violations` for dumping the findings into the test output. Use those first. The custom version below is still worth reading, because it shows what the built-in steps do under the hood and how you would tune the failure rules if your project needs different thresholds:

```javascript
// tests/step-definitions/custom-a11y.js
const { Then } = require('@cucumber/cucumber');
const assert = require('assert');

Then('the page should pass accessibility checks', async function () {
  // Inject axe-core
  await this.page.addScriptTag({
    url: 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.7.0/axe.min.js'
  });

  // Run axe
  const results = await this.page.evaluate(async () => {
    return await axe.run();
  });

  const violations = results.violations;
  if (violations.length > 0) {
    const report = violations.map(v =>
      `${v.impact}: ${v.description} (${v.nodes.length} instances)`
    ).join('\n');

    assert.fail(`Accessibility violations found:\n${report}`);
  }
});

Then('the page should have no critical accessibility issues', async function () {
  await this.page.addScriptTag({
    url: 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.7.0/axe.min.js'
  });

  const results = await this.page.evaluate(async () => {
    return await axe.run();
  });

  const critical = results.violations.filter(
    v => v.impact === 'critical' || v.impact === 'serious'
  );

  if (critical.length > 0) {
    const report = critical.map(v =>
      `[${v.impact}] ${v.description}\n  ${v.helpUrl}`
    ).join('\n\n');

    assert.fail(`Critical accessibility violations:\n${report}`);
  }
});
```

```gherkin
Feature: Accessibility Audit
  @a11y
  Scenario: Homepage passes accessibility checks
    Given I am on homepage
    And wait for the page to load
    Then the page should pass an accessibility audit at level "AA"

  @a11y
  Scenario: Login page has no critical issues
    Given I am on homepage
    When I go to "/login"
    And wait for the page to load
    Then the page should have no critical accessibility violations

  @a11y
  Scenario: Dashboard passes accessibility checks
    Given I am a logged in user with the "admin" user
    When I go to "/admin/dashboard"
    And wait for AJAX to finish
    And wait for the page to load
    Then the page should have no critical accessibility violations
```

**How It Works:**

axe-core is the industry standard accessibility testing engine. It checks for:

```
axe-core Checks:
┌──────────────────────────────────────────────────────────────────┐
│ Category          │ What It Checks                               │
├──────────────────────────────────────────────────────────────────┤
│ Color contrast    │ Text vs background ratio (WCAG AA/AAA)       │
│ Alt text          │ Images, icons, inputs have labels             │
│ ARIA roles        │ Valid roles, required properties              │
│ Form labels       │ Inputs associated with labels                 │
│ Headings          │ Correct heading hierarchy (h1→h2→h3)        │
│ Landmarks         │ Page has navigation, main, etc.              │
│ Links             │ Link text is descriptive (not "click here")  │
│ Tables            │ Tables have headers and captions              │
│ Focus             │ Focus indicators visible                      │
│ Language          │ Page has lang attribute                       │
└──────────────────────────────────────────────────────────────────┘

Impact Levels:
  critical  → Blocks entire user groups (e.g., no alt text)
  serious   → Significant barrier (e.g., missing form labels)
  moderate  → Reduces usability (e.g., heading hierarchy skip)
  minor     → Inconvenience (e.g., redundant roles)
```

webship-js's built-in audit steps run this engine against the live, rendered page -- which matters for contrast specifically, because the real ratio depends on computed styles, inherited backgrounds, and overlays that only exist at runtime. The level parameter ("A", "AA", "AAA") selects which WCAG rule set applies; the no-critical-violations step filters by impact instead, which lets a legacy page run a looser gate while new work holds the full line. And because the engine ships bundled with the framework, the audit works on air-gapped CI runners where the CDN injection in the custom version above would fail outright -- one more reason to prefer the built-ins. When a run goes red, `Then print accessibility violations` writes each finding with its impact and help URL into the test output, which lands in the JSON report and therefore in the HTML report your team actually reads.

**At scale:** One audit step, the same level "AA" string, in every project -- that consistency is what turns accessibility from a per-project opinion into a portfolio-wide number. Tag the audits @a11y, run them in every pipeline, and stamp the result into report metadata (`--metadata A11y=AA`) so the cross-project dashboard can show it. Fair warning: the first nightly run across 20 legacy sites will be ugly. Gate on critical violations first, ratchet each project up to the full AA audit as it cleans up, and the dashboard shows your PMO an improving line instead of a wall of red nobody acts on.

---

## Recipe 158: Testing Screen Reader Content

**Problem:** Your form saves and a green toast says so -- visually. A screen reader user hears nothing, waits, and submits again, creating a duplicate record. The cues sighted users get for free (a toast, a highlight, a spinner) have to be deliberately wired into assistive tech with aria-live regions and skip links, and deliberate wiring is exactly what disappears silently in refactors.

**Solution:**

```javascript
// tests/step-definitions/custom-sr.js
const { Then } = require('@cucumber/cucumber');
const assert = require('assert');

Then('there should be a skip link', async function () {
  const skipLink = this.page.locator('a[href="#main-content"], a.skip-link, a.skip-to-content');
  const count = await skipLink.count();
  assert(count > 0, 'No skip-to-content link found');
});

Then('the {string} element should have aria-live {string}', async function (selector, value) {
  const ariaLive = await this.page.locator(selector).first().getAttribute('aria-live');
  assert.strictEqual(ariaLive, value,
    `Expected aria-live="${value}", got "${ariaLive}"`);
});
```

```gherkin
Feature: Screen Reader Accessibility
  Scenario: Skip link is available
    Given I am on homepage
    Then there should be a skip link

  Scenario: Status messages are announced
    Given I am a logged in user with the "admin" user
    When I go to "/settings"
    Then the ".status-messages" element should have aria-live "polite"
    When I press "Save"
    And wait for AJAX to finish
    Then I should see "Settings saved"
```

**How It Works:**

The skip-link step queries the three common conventions in one locator and passes if any exists -- though webship-js also ships a structural skip-link check built in, so reach for the custom version only when your projects use a selector the built-in does not know. The aria-live assertion is the one to internalize: a region marked `aria-live="polite"` makes assistive tech announce any text injected into it, which is the only way your "Settings saved" toast reaches a screen reader at all. Note the ordering in the scenario -- the attribute is checked *before* triggering the save, on purpose. The live region must exist in the DOM before the message arrives, because screen readers do not announce content placed in regions created after the fact. That ordering bug is the classic way live regions fail while looking perfectly correct in the DOM inspector.

---

## Accessibility Testing Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│ Accessibility Testing Checklist for webship-js                   │
│                                                                  │
│ Automated (axe-core):                                           │
│   □ Color contrast ratios meet WCAG AA                          │
│   □ All images have alt text                                     │
│   □ Form inputs have associated labels                           │
│   □ Page has proper landmark roles                               │
│   □ Heading hierarchy is correct (no skipped levels)             │
│   □ Links have descriptive text                                  │
│   □ ARIA attributes are valid                                    │
│   □ Language attribute is set on <html>                          │
│                                                                  │
│ Semi-Automated (custom steps):                                   │
│   □ Tab order follows visual order                               │
│   □ All interactive elements are keyboard accessible             │
│   □ Focus moves correctly after actions (modals, errors)         │
│   □ Skip link is present and functional                          │
│   □ Modal focus trapping works                                   │
│   □ Dynamic content updates announce to screen readers           │
│                                                                  │
│ Manual Testing (complement automated):                           │
│   □ Screen reader can navigate the full application              │
│   □ Zoom to 200% does not break layout                          │
│   □ No information is conveyed by color alone                    │
│   □ Animation can be paused or disabled                          │
│   □ Touch targets are large enough on mobile                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- Accessibility is not optional -- it is a core quality attribute
- WCAG 2.1 defines four principles: Perceivable, Operable, Understandable, Robust
- Keyboard navigation testing ensures Tab, Enter, Escape work correctly
- Focus management testing verifies focus moves after modals, errors, navigation
- ARIA labels and roles must be present for screen reader users
- axe-core provides automated accessibility auditing with severity levels
- Run accessibility checks on every page (homepage, login, dashboard, forms)
- Use `@a11y` tags to organize accessibility tests
- Color contrast must meet WCAG AA minimums (4.5:1 normal, 3:1 large)
- Automated testing catches ~30-50% of accessibility issues; manual testing covers the rest
- Skip links, aria-live regions, and proper landmarks are testable with custom steps

This concludes Part III: API and Advanced Testing.

Next: The PlaywrightWorld Deep Dive -- understanding the architecture that powers webship-js.

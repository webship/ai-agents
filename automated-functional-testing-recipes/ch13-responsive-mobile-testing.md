# Chapter 13: Responsive and Mobile Testing

---

More than half of all web traffic comes from mobile devices. Your application must look and function correctly on screens ranging from a 375px-wide iPhone SE to a 2560px-wide ultrawide monitor. Responsive design is not optional -- it is a core requirement.

This chapter shows you how to test responsive behavior, verify mobile layouts, and catch breakpoint-related bugs.

---

## Understanding Responsive Breakpoints

```
┌────────────────────────────────────────────────────────────────────┐
│ Common CSS Breakpoints                                             │
│                                                                    │
│  375px      576px      768px      992px       1200px     1400px    │
│    │          │          │          │           │          │       │
│    ▼          ▼          ▼          ▼           ▼          ▼       │
│    XS         SM         MD         LG          XL         XXL     │
│                                                                    │
│  ┌──┐     ┌────┐    ┌──────┐   ┌────────┐  ┌──────────┐            │
│  │  │     │    │    │      │   │        │  │          │            │
│  │  │     │    │    │      │   │        │  │          │            │
│  │  │     │    │    │      │   │        │  │          │            │
│  └──┘     └────┘    └──────┘   └────────┘  └──────────┘            │
│  Phone    Phone     Tablet     Desktop     Desktop                 │
│  portrait landscape                        Large                   │
│                                                                    │
│  Layout:  Layout:   Layout:    Layout:     Layout:                 │
│  1 col    1-2 col   2 col      3 col       3 col + sidebar         │
│  Stack    Cards     Sidebar    Full nav    Full nav                │
│  Hamburger Hamburger Full nav  Full nav    Full nav                │
└────────────────────────────────────────────────────────────────────┘
```

webship-js 2.0 ships this table as a named breakpoints registry in its worldParameters: xs 375x667, sm 576x800, md 768x1024, lg 992x768, xl 1200x900 (the default), xxl 1400x900, and xxxl 1920x1080. You can replace the whole table per project -- or per CI run -- with the WEBSHIP_SELECTORS_BREAKPOINTS environment variable (JSON). Keep your test viewports aligned with the breakpoints your CSS actually uses; testing at 800px when your CSS switches at 768px proves nothing about either side of the switch.

---

## Recipe 143: Testing at Different Viewport Sizes

**Problem:** A designer opens the site on her phone and files a bug: the pricing cards overlap at 375px wide. Your suite is green -- because every scenario ran at whatever width the CI browser happened to open with. Until you pin viewports in your tests, "responsive design works" is an assumption, not a verified behavior, and the first person to verify it is a user.

**Solution:**

You can set the viewport through Playwright's context options. Create custom step definitions for viewport control:

```javascript
// tests/step-definitions/custom-viewport.js
const { Given, When } = require('@cucumber/cucumber');

When('I resize the browser to {int} by {int}', async function (width, height) {
  await this.page.setViewportSize({ width, height });
});

Given('I am on a mobile device', async function () {
  await this.page.setViewportSize({ width: 375, height: 667 });
});

Given('I am on a tablet', async function () {
  await this.page.setViewportSize({ width: 768, height: 1024 });
});

Given('I am on a desktop', async function () {
  await this.page.setViewportSize({ width: 1920, height: 1080 });
});
```

```gherkin
Feature: Responsive Layout
  Scenario: Mobile layout shows hamburger menu
    Given I am on a mobile device
    And I am on homepage
    Then I should not see "Products"
    And I should not see "About Us"
    # Hamburger menu icon should be visible instead
    When I press "Menu"
    Then I should see "Products"
    And I should see "About Us"

  Scenario: Desktop layout shows full navigation
    Given I am on a desktop
    And I am on homepage
    Then I should see "Products"
    And I should see "About Us"
    And I should see "Contact"

  Scenario: Tablet shows adapted layout
    Given I am on a tablet
    And I am on homepage
    Then I should see "Products"
    And I should see "About Us"

  Scenario: Test specific viewport
    When I resize the browser to 1024 by 768
    Given I am on homepage
    Then I should see "Products"
```

**How It Works:**

`page.setViewportSize()` resizes the rendering surface, re-evaluates every CSS media query, and fires the `resize` event -- the same thing that happens when a real user rotates a phone. webship-js's playwright.config.ts ships with `viewport: null`, meaning the page follows the launched window size; calling `setViewportSize()` overrides that for the current page. Set it before navigation (as these steps do) so the page loads with its media queries already evaluated at the target width -- some layout JavaScript only measures the screen once, on load.

Notice that the mobile and tablet steps match webship-js's own breakpoints registry: 375x667 is xs, 768x1024 is md. Reusing the registry's numbers instead of inventing your own keeps "mobile" meaning the same thing in your custom steps, your selector files, and your CSS. And there is no sleep after pressing "Menu" -- webship-js runs smartSettle after every state-changing step, waiting for pending timers to drain and the DOM to stay quiet for 250ms, so the menu's open animation finishes before the next assertion fires.

**At scale:** Do not let each project define its own idea of "mobile". Keep one breakpoints JSON in a shared config package and inject it with WEBSHIP_SELECTORS_BREAKPOINTS in CI. When the design system moves a breakpoint, forty suites move with it -- one env var change in the pipeline template instead of forty pull requests, each reviewed by someone who has to re-derive why 768 became 800.

---

## Recipe 144: Testing Mobile Navigation (Hamburger Menu)

**Problem:** The hamburger menu is the kind of component that breaks quietly. A z-index bump for a promo banner, and the menu now opens behind the hero image; a refactor renames the toggle class and the button does nothing. Nobody on the team notices, because the team lives on desktop -- the first report comes from a customer on a phone who cannot reach the Products page at all.

**Solution:**

```gherkin
Feature: Mobile Navigation
  Background:
    Given I am on a mobile device

  Scenario: Hamburger menu opens and closes
    Given I am on homepage
    When I press "Menu"
    Then I should see "Home"
    And I should see "Products"
    And I should see "About"
    And I should see "Contact"

    When I press "Menu"
    Then I should not see "Products"

  Scenario: Navigate via hamburger menu
    Given I am on homepage
    When I press "Menu"
    And I click "Products"
    And wait for AJAX to finish
    Then I should see "Our Products"

  Scenario: Menu closes after navigation
    Given I am on homepage
    When I press "Menu"
    And I click "About"
    And wait for AJAX to finish
    Then I should see "About Us"
    # Menu should auto-close after navigation
    And I should not see "Contact"
```

**How It Works:**

Menu open and close are usually animated, and animations are where naive suites reach for sleeps. webship-js does not need them. After every state-changing step, an AfterStep hook runs smartSettle(page, 1500), which waits until in-flight fetch/XHR calls hit zero (tracked by window.__webshipAjaxCount), wrapped setTimeout timers drain (window.__webshipPendingTimers), and a MutationObserver reports the DOM quiet for at least 250ms (window.__webshipLastMutation). A 300ms slide-in that toggles classes is exactly the kind of activity those trackers see, so by the time `Then I should see "Home"` runs, the menu is genuinely open. This is why webship-js 2.0 ships no static sleep step at all -- a `wait 1s` would only paper over timing the framework already handles.

The third scenario checks the detail teams forget: on single-page apps, navigating from inside the menu often leaves the menu hanging open over the new page. Asserting `I should not see "Contact"` after navigation pins that behavior down before a user finds it.

**At scale:** Tag these scenarios @mobile and make the tag mean the same thing in every repo. When the CI farm runs nightly across 20 projects, `--tags @mobile` gives one comparable answer to "does mobile navigation work everywhere?" -- and when a shared navbar component ships a regression, one report tells you which products consumed the broken version, instead of twenty separate bug triages reaching the same conclusion by Friday.

---

## Recipe 145: Testing Responsive Images

**Problem:** Your team added `srcset` images to cut mobile load time, and the metric moved the wrong way -- phones were still downloading the 2400px hero. Nothing functional broke, so no assertion failed; the only symptoms were a slow page and a confused performance review two sprints later. Responsive images fail silently, which makes them exactly the kind of thing worth a test.

**Solution:**

```gherkin
Feature: Responsive Images
  Scenario: Mobile shows optimized images
    Given I am on a mobile device
    When I go to "/products"
    And wait for AJAX to finish
    Then I should see "Products"
    # Verify content loads (images load as part of page)
    And wait for the page to load

  Scenario: Desktop shows full-size images
    Given I am on a desktop
    When I go to "/products"
    And wait for AJAX to finish
    And wait for the page to load
    Then I should see "Products"
```

**Note:** Testing that the correct image _variant_ loaded (small vs large) typically requires checking the rendered image dimensions or the `currentSrc` property. This needs a custom step:

```javascript
// tests/step-definitions/custom-responsive-image.js
const { Then } = require('@cucumber/cucumber');
const assert = require('assert');

Then('the {string} image should be narrower than {int} pixels', async function (alt, maxWidth) {
  const img = this.page.locator(`img[alt="${alt}"]`).first();
  const box = await img.boundingBox();
  assert(box.width < maxWidth,
    `Image "${alt}" is ${box.width}px wide, expected less than ${maxWidth}px`);
});
```

**How It Works:**

`boundingBox()` returns the rendered size in CSS pixels after layout -- what the user actually gets at this viewport, and a fast proxy for "the right variant loaded". To verify the variant directly, read `currentSrc` (not `src`) inside `page.evaluate()`: `currentSrc` is the URL the browser actually chose from the srcset after applying device pixel ratio and the `sizes` attribute. One caution -- assert on dimensions or the chosen URL, never on bytes downloaded. Browsers may legitimately reuse a larger cached variant after a resize, and a byte-level assertion turns that optimization into a flaky failure you will chase for a week.

---

## Recipe 146: Testing Media Query Breakpoints

**Problem:** A developer "tidies up" a media query from `max-width: 768px` to `max-width: 767.98px` -- or flips a `min-width` to a `max-width` -- and the layout now has a one-pixel band where the page shows neither the hamburger nor the full menu. iPad users in portrait, who sit at exactly 768px, get the broken state. You will never catch this by testing "mobile" and "desktop"; you catch it by testing the boundaries themselves.

**Solution:**

```gherkin
Feature: Breakpoint Testing
  Scenario Outline: Layout adapts at breakpoints
    When I resize the browser to <width> by 800
    Given I am on homepage
    Then I should see "<expected_element>"

    Examples:
      | width | expected_element    |
      | 375   | Menu                |
      | 576   | Menu                |
      | 768   | Products            |
      | 992   | Products            |
      | 1200  | Products            |

  Scenario: Sidebar appears on large screens
    When I resize the browser to 1200 by 800
    Given I am on homepage
    When I go to "/blog"
    And wait for AJAX to finish
    Then I should see "Recent Posts"
    And I should see "Categories"

  Scenario: Sidebar hidden on small screens
    When I resize the browser to 375 by 667
    Given I am on homepage
    When I go to "/blog"
    And wait for AJAX to finish
    Then I should see "Recent Posts"
    And I should not see "Categories"
```

**How It Works:**

The Scenario Outline tests the same page at each breakpoint, verifying that the expected elements are visible. This catches CSS bugs where a breakpoint is slightly off (e.g., `max-width: 767px` instead of `max-width: 768px`).

Each Examples row runs as its own scenario with a fresh World and a fresh browser -- the resize never leaks between rows, and a failure at 768px shows up as its own line in the report rather than burying itself inside a loop. That is also why the resize step comes before navigation in each row: the page loads with its media queries already evaluated at the target width, exactly as it would on a real tablet. When a row fails, the report hands you the exact width, which usually points straight at the off-by-one in the CSS.

If you want to be even stricter, test one pixel either side of each boundary (767 and 768, 991 and 992). Boundary pairs are where every breakpoint bug lives.

```
Breakpoint Test Matrix:
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Element  │  375px   │  768px   │  992px   │ 1200px   │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│ Nav Menu │ Hamburger│ Full     │ Full     │ Full     │
│ Sidebar  │ Hidden   │ Hidden   │ Visible  │ Visible  │
│ Grid Cols│ 1        │ 2        │ 3        │ 4        │
│ Footer   │ Stacked  │ 2 cols   │ 4 cols   │ 4 cols   │
└──────────┴──────────┴──────────┴──────────┴──────────┘
```

**At scale:** Breakpoint matrices multiply fast: 5 widths x 3 browsers x 20 projects is 300 browser launches before you have tested a single behavior, and your DevOps team will notice the nightly bill. Run the full matrix under its own tag (@breakpoints) on the nightly schedule, keep pull-request runs to the default xl viewport plus one mobile width, and stamp the report with `--metadata Viewport=375` so the cross-project dashboard can slice failures by width instead of someone reading forty reports.

---

## Recipe 147: Testing Touch-Friendly Elements

**Problem:** Support keeps logging the same complaint from mobile users: "the Save button does not work." It works fine -- it is 28 pixels tall and sits 4 pixels from a Cancel link, so thumbs miss it or hit Cancel instead. Apple's HIG says 44x44 CSS pixels minimum, Material Design says 48x48dp, and nobody on the team measures either until the complaints arrive.

**Solution:**

```javascript
// tests/step-definitions/custom-touch-target.js
const { Then } = require('@cucumber/cucumber');
const assert = require('assert');

Then('the {string} button should be at least {int}px tall', async function (text, minHeight) {
  const button = this.page.getByRole('button', { name: text }).first();
  const box = await button.boundingBox();
  assert(box.height >= minHeight,
    `Button "${text}" is ${box.height}px tall, minimum is ${minHeight}px`);
});
```

```gherkin
Feature: Touch-Friendly UI
  Background:
    Given I am on a mobile device

  Scenario: Buttons are large enough to tap
    Given I am on homepage
    When I go to "/login"
    Then the "Sign In" button should be at least 44px tall

  Scenario: Mobile form fields are accessible
    Given I am on homepage
    When I go to "/contact"
    Then I should see "Name"
    And I should see "Email"
    And I should see "Message"
    # Fields should be tappable and fillable on mobile
    When I fill in "Name" with "Rajab"
    And I fill in "Email" with "rajab@example.com"
    And I fill in "Message" with "Mobile message"
    And I press "Send"
    And wait for AJAX to finish
    Then I should see "Message sent"
```

**How It Works:**

`getByRole('button', { name: text })` resolves the button through the accessibility tree -- the same mechanism behind webship-js's built-in `When I click the "Save" button` step -- so it matches `<button>`, `input[type=submit]`, and ARIA `role="button"` elements alike. `boundingBox()` then returns the element's rendered size in CSS pixels at the current viewport. That last part is the point of the Background line: a button can be a comfortable 48px on desktop and shrink below the threshold at 375px, where responsive padding collapses. Measure at the viewport where thumbs actually do the tapping.

The second scenario looks redundant -- fill a form, like Chapter 6 -- but running it at mobile width catches a class of bug desktop tests cannot: fields hidden behind a sticky footer, labels truncated into ambiguity, a Send button pushed off-screen by the on-page keyboard layout.

---

## Responsive Testing Quick Reference

```
┌───────────────────────────────────────────────────────────────────┐
│ What You Want                    │ Approach                       │
├───────────────────────────────────────────────────────────────────┤
│ Test mobile layout               │ Given I am on a mobile device  │
│ Test tablet layout               │ Given I am on a tablet         │
│ Test desktop layout              │ Given I am on a desktop        │
│ Test specific viewport           │ When I resize the browser to   │
│                                  │   WxH                          │
│ Test hamburger menu              │ Press "Menu" → verify items    │
│ Test breakpoint transitions      │ Scenario Outline with widths   │
│ Test touch targets               │ Custom step: bounding box      │
│ Test mobile device emulation     │ Playwright devices config      │
└───────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- More than half of web traffic is mobile -- responsive testing is essential
- Custom viewport steps let you test at any screen size
- Test breakpoint boundaries where layout changes occur
- Mobile navigation (hamburger menus) needs explicit open/close testing
- Responsive images and media queries are testable through content visibility
- Touch target size validation ensures mobile usability
- Scenario Outlines with viewport widths efficiently test all breakpoints
- Playwright's device profiles simulate real mobile devices
- Combine viewport testing with browser testing for comprehensive coverage

Next: Screenshot and Visual Testing -- capturing evidence and detecting visual regressions.

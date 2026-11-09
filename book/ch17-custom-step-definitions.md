# Chapter 17: Writing Custom Step Definitions

---

webship-js gives you a comprehensive library of built-in steps: navigation, forms, assertions, waits, modals, tables, API testing. But every project has unique features that no generic library can anticipate. Maybe your app has a drag-and-drop kanban board, a custom date picker, a multi-select autocomplete, or a real-time chat widget.

This is where custom step definitions come in. You write the Gherkin that describes your feature in business language, then implement the automation behind it.

This chapter teaches you everything about writing custom step definitions: from your first simple step to advanced patterns with data tables, doc strings, and domain-specific vocabularies.

---

## Step Definition Anatomy

```
┌──────────────────────────────────────────────────────────────────┐
│ How a Gherkin Step Becomes an Action                             │
│                                                                  │
│  Feature File:                                                   │
│  ┌──────────────────────────────────────────────┐               │
│  │ When I add "Laptop" to the cart              │               │
│  └──────────────────────┬───────────────────────┘               │
│                         │ Cucumber matches                       │
│                         │ step text to pattern                   │
│                         ▼                                        │
│  Step Definition:                                                │
│  ┌──────────────────────────────────────────────┐               │
│  │ When('I add {string} to the cart',           │               │
│  │   async function (product) {                 │               │
│  │     // product = "Laptop"                    │               │
│  │     await this.page                          │               │
│  │       .getByText(product)                    │               │
│  │       .locator('.. >> button')               │               │
│  │       .click();                              │               │
│  │   }                                          │               │
│  │ );                                           │               │
│  └──────────────────────────────────────────────┘               │
│                         │                                        │
│                         │ Step runs                               │
│                         ▼                                        │
│  Browser Action:                                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │ Playwright finds "Laptop" text, then clicks  │               │
│  │ the "Add to Cart" button next to it          │               │
│  └──────────────────────────────────────────────┘               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Every step definition has three parts:
1. **Keyword** (`Given`, `When`, `Then`)
2. **Pattern** (string or regex that matches the Gherkin text)
3. **Function** (the code that runs when the pattern matches)

---

## Cucumber Expressions vs Regular Expressions

webship-js uses both. Here is when to use each:

```
┌──────────────────────────────────┬──────────────────────────────┐
│ Cucumber Expressions             │ Regular Expressions          │
│                                  │                              │
│ When('I add {string} to cart',   │ When(/^I add "([^"]*)" to   │
│   function (product) { ... }     │   cart$/, function (product) │
│ );                               │   { ... }                    │
│                                  │ );                           │
│                                  │                              │
│ Simpler, more readable           │ More powerful, flexible      │
│ Built-in types: {string},       │ Full regex patterns           │
│   {int}, {float}, {word}        │ Capture groups               │
│ Custom parameter types           │ Optional parts (?:...)?      │
│                                  │ Alternation (a|b)            │
│                                  │                              │
│ USE WHEN:                        │ USE WHEN:                    │
│ Simple parameter extraction     │ Complex patterns needed      │
│ New step definitions             │ Optional words/phrases       │
│ Readable > flexible              │ Matching flexibility needed  │
└──────────────────────────────────┴──────────────────────────────┘
```

---

## Recipe 162: Your First Custom Given Step

**Problem:** Your login flow is unusual -- maybe it is SSO, maybe it has a custom role picker -- and every scenario in the suite starts with the same six lines of clicking and filling. The day a developer renames the "Sign In" button, forty scenarios fail at the same step and the new QA hire spends an afternoon fixing forty copies of the same Gherkin. One custom Given step would have made that a one-line fix.

**Solution:**

```javascript
// tests/step-definitions/custom-setup.js
const { Given } = require('@cucumber/cucumber');

// Cucumber Expression
Given('I am a logged in user with the {string} user', async function (role) {
  const users = {
    admin: { email: 'admin@example.com', password: 'AdminPass123' },
    editor: { email: 'editor@example.com', password: 'EditorPass123' },
    customer: { email: 'customer@example.com', password: 'CustPass123' },
  };

  const user = users[role];
  if (!user) throw new Error(`Unknown user role: ${role}`);

  await this.page.goto(this.launchUrl + '/login');
  await this.page.waitForLoadState('networkidle');
  await this.page.getByLabel('Email').fill(user.email);
  await this.page.getByLabel('Password').fill(user.password);
  await this.page.getByRole('button', { name: 'Sign In' }).click();
  await this.page.waitForLoadState('networkidle');
});
```

```gherkin
Feature: Admin Dashboard
  Scenario: Admin sees dashboard
    Given I am a logged in user with the "admin" user
    When I go to "/admin/dashboard"
    Then I should see "Dashboard"
```

**How It Works:** Cucumber matches the Gherkin line against the Cucumber Expression `'I am a logged in user with the {string} user'`, extracts `"admin"` into the `role` parameter, and calls your function with `this` bound to the World -- the same object the 411 built-in webship-js steps use. That is why `this.page` and `this.launchUrl` are already there: the built-in Before hook created the browser context and stored them before your step ran. The user map keeps credentials in one place; the `throw` on an unknown role turns a typo in a feature file into a clear error instead of a confusing login failure. Because your custom step lives in `tests/step-definitions/`, the default cucumber.js profile picks it up automatically -- no registration, no imports in the feature file.

**At scale:** When you run twenty projects, do not hardcode the user map per project. Read it from an environment-driven JSON file (`TEST_USERS_FILE`) or environment variables, so the same step definition file can be copied -- or better, shared as a small npm package -- across every suite. Your CI then injects per-environment credentials as secrets, and no password ever lives in a repository. Also consider the built-in auth-state steps (save/restore storageState) so login happens once per run, not once per scenario -- across 40 nightly suites that is real CI minutes saved.

---

## Recipe 163: Your First Custom When Step

**Problem:** Your product pages have a card layout where the "Add to Cart" button only makes sense relative to the product it belongs to. The generic click steps can find *a* button, but not *the* button next to "Laptop Pro" -- so testers start writing brittle CSS like `.product-list > div:nth-child(3) button`, and the suite breaks the first time marketing reorders the product grid. You need action steps that express the relationship: this product, its button.

**Solution:**

```javascript
// tests/step-definitions/custom-actions.js
const { When } = require('@cucumber/cucumber');

When('I add {string} to the cart', async function (productName) {
  const product = this.page.locator('.product-card')
    .filter({ hasText: productName });
  await product.locator('button', { hasText: 'Add to Cart' }).click();
});

When('I remove {string} from the cart', async function (productName) {
  const item = this.page.locator('.cart-item')
    .filter({ hasText: productName });
  await item.locator('button', { hasText: 'Remove' }).click();
});

When('I set the quantity of {string} to {int}', async function (productName, quantity) {
  const item = this.page.locator('.cart-item')
    .filter({ hasText: productName });
  await item.locator('input[type="number"]').fill(String(quantity));
});
```

```gherkin
Feature: Shopping Cart
  Scenario: Add products and update quantities
    Given I am on homepage
    When I go to "/products"
    And I add "Laptop Pro" to the cart
    And I add "Wireless Mouse" to the cart
    And I go to "/cart"
    Then I should see "Laptop Pro"
    And I should see "Wireless Mouse"
    When I set the quantity of "Laptop Pro" to 2
    And I remove "Wireless Mouse" from the cart
    Then I should see "Laptop Pro"
    And I should not see "Wireless Mouse"
```

**How It Works:** The pattern here is *scope, then act*. `locator('.product-card').filter({ hasText: productName })` narrows the page to the one card containing the product name, and the second `locator()` call searches only inside that card. The position of the card on the page no longer matters -- reorder the grid, add a banner, paginate the list, and the step still finds the right button. Because these are `When` steps that change state, webship-js's AfterStep hook fires `smartSettle(page, 1500)` after each one: it waits for `window.__webshipAjaxCount` to hit zero, pending timers to drain, and the DOM to stay quiet for 250ms before the next step runs. That is why there is no explicit wait after the click -- the cart badge update the next assertion depends on has already settled.

**At scale:** Notice that all three steps share the same two-line scoping idiom. When this pattern appears in your third project, extract a helper -- `cartItem(page, name)` -- into a shared support module so every suite scopes the same way. Teams that skip this end up with five slightly different `filter({ hasText })` variants per project, and nobody knows which one handles the edge cases.

---

## Recipe 164: Your First Custom Then Step

**Problem:** "Then I should see '2'" passes when the cart badge says 2 -- and also when the year 2026 appears in the footer. Generic text assertions are too loose for business rules like cart counts and order totals, and when one fails, the error message ("expected to see 2") tells the reviewer nothing about *what* was wrong. The Tuesday before release, a vague failure like that costs an hour of someone re-running the scenario in headed mode just to understand the message.

**Solution:**

```javascript
// tests/step-definitions/custom-assertions.js
const { Then } = require('@cucumber/cucumber');
const assert = require('assert');

Then('the cart should have {int} items', async function (expectedCount) {
  const badge = this.page.locator('.cart-badge, .cart-count').first();
  const text = await badge.textContent();
  const count = parseInt(text, 10);
  assert.strictEqual(count, expectedCount,
    `Expected ${expectedCount} items in cart, found ${count}`);
});

Then('the total should be {string}', async function (expectedTotal) {
  const total = this.page.locator('.cart-total, .order-total').first();
  const text = await total.textContent();
  assert(text.includes(expectedTotal),
    `Expected total to contain "${expectedTotal}", got "${text}"`);
});

Then('the {string} notification should appear', async function (type) {
  const selector = `.notification-${type}, .alert-${type}, .toast-${type}`;
  const notification = this.page.locator(selector).first();
  await notification.waitFor({ state: 'visible', timeout: 5000 });
});
```

```gherkin
Feature: Cart Totals
  Scenario: Cart calculates total correctly
    Given I am a logged in user with the "customer" user
    When I go to "/products"
    And I add "Laptop Pro" to the cart
    And I add "USB Cable" to the cart
    And I go to "/cart"
    Then the cart should have 2 items
    And the total should be "$1,008.99"
    And the "success" notification should appear
```

**How It Works:** Each assertion targets the exact element that carries the business meaning -- the badge, the total, the notification -- instead of searching the whole page for a string. The custom failure messages are the real payoff: `Expected 2 items in cart, found 3` tells a reviewer reading the HTML report what broke without opening a browser. The notification step uses `waitFor({ state: 'visible', timeout: 5000 })`, which polls rather than checking once -- the same web-first philosophy as the built-in `Then "sel" should be visible` step (100ms polling, 5s default). One caveat: `textContent()` reads immediately, so for values that update via AJAX, lean on the AfterStep smartSettle that already ran after your last action -- by the time a `Then` step executes, ajax count is zero and the DOM has been quiet for 250ms.

**Pro Tip:** Write the failure message before you write the assertion. If you cannot phrase what a failure should say, you have not decided what the step actually checks.

---

## Recipe 165: Steps with String Parameters

**Problem:** You wrote `When I search for laptops` as a step, and now product owners want to reuse it for "wireless mouse", "USB-C hub", and a dozen other queries. Without parameters you would copy the step definition once per search term -- the exact duplication that makes a suite rot. You need one pattern that captures whatever sits between the quotes.

**Solution:**

```javascript
// Cucumber Expression: {string} captures quoted text
When('I search for {string}', async function (query) {
  await this.page.getByPlaceholder('Search').fill(query);
  await this.page.keyboard.press('Enter');
});

// Regex: ([^"]*) captures quoted text
When(/^I search for "([^"]*)"$/, async function (query) {
  await this.page.getByPlaceholder('Search').fill(query);
  await this.page.keyboard.press('Enter');
});
```

Both patterns match: `When I search for "laptops"`

**How It Works:** With the Cucumber Expression, `{string}` matches anything inside double quotes and hands it to your function already unquoted. The regex version does the same job with the capture group `([^"]*)` -- "any run of characters that is not a quote" -- but you maintain the anchors (`^`, `$`) and escaping yourself. One subtle difference matters in practice: `{string}` also matches single-quoted text, so `When I search for 'laptops'` works too; the regex above only matches double quotes. Pick Cucumber Expressions for new steps and save regex for cases that genuinely need it (Recipe 170). A warning from maintaining suites across many projects: if both patterns are loaded at once, Cucumber throws an ambiguous-step error -- never keep two definitions that match the same text.

---

## Recipe 166: Steps with Integer Parameters

**Problem:** A search results assertion needs to work for 3 results today and 12 results after the next content import. Hardcoding the count into the step text means editing step definitions every time the data changes -- and worse, someone will eventually compare the string `"12"` to the number `12` and spend twenty minutes staring at an assertion that "should obviously pass".

**Solution:**

```javascript
// Cucumber Expression: {int} captures an integer
Then('I should see {int} results', async function (count) {
  const results = await this.page.locator('.search-result').count();
  assert.strictEqual(results, count);
});

When('I wait {int} seconds', async function (seconds) {
  await this.page.waitForTimeout(seconds * 1000);
});

// Regex: (\d+) captures digits
Then(/^I should see (\d+) results$/, async function (count) {
  const results = await this.page.locator('.search-result').count();
  assert.strictEqual(results, parseInt(count, 10));
});
```

Note: With regex, captured values are always strings -- you need `parseInt()`. With Cucumber Expressions, `{int}` automatically converts to a number.

**How It Works:** `{int}` does two things: it matches a run of digits (with optional minus sign) in the Gherkin text, and it converts the match to a JavaScript number before your function sees it. That conversion is what makes `assert.strictEqual(results, count)` safe -- `locator().count()` returns a number, and strict equality between a number and a string always fails. The regex version skips the conversion, which is exactly the trap the `parseInt(count, 10)` call defuses. And a word about that `I wait {int} seconds` example: it shows the mechanics of `{int}`, but treat it as a debugging tool only. webship-js has a hard rule -- no static sleeps. Every built-in step settles through smartSettle (ajax count zero, timers drained, DOM quiet for 250ms), so a scenario that needs a hand-written sleep is usually telling you an assertion should be a polling web-first check instead.

---

## Recipe 167: Steps with Data Tables

**Problem:** A registration form has nine fields. Writing nine `When I fill in` lines per scenario works, but the third time a product owner reviews the feature file, their eyes glaze over before line five -- and when the form gains a tenth field, you edit every scenario that touches it. A table turns nine noisy lines into one readable block that a BA can review, edit, and even extend without asking engineering.

**Solution:**

```javascript
const { When, Then } = require('@cucumber/cucumber');
const assert = require('assert');

// Key-value table
When('I fill in the profile form with:', async function (table) {
  for (const [field, value] of table.raw()) {
    await this.page.getByLabel(field).fill(value);
  }
});

// Multi-column table
Then('the user list should contain:', async function (table) {
  const rows = table.hashes(); // [{name: "John", role: "Admin"}, ...]
  for (const row of rows) {
    const tr = this.page.locator('tr').filter({ hasText: row.name });
    const text = await tr.textContent();
    assert(text.includes(row.role),
      `Expected "${row.name}" to have role "${row.role}"`);
  }
});
```

```gherkin
Feature: User Profile
  Scenario: Fill profile form
    When I fill in the profile form with:
      | First Name | Rajab         |
      | Last Name  | Natshah       |
      | Email      | r@example.com |
      | Phone      | +1234567890   |

  Scenario: Verify user list
    Then the user list should contain:
      | name       | role   | status |
      | John Smith | Admin  | Active |
      | Jane Doe   | Editor | Active |
```

**Data Table Methods:**

```
┌────────────────────┬──────────────────────────────────────────┐
│ Method             │ Returns                                  │
├────────────────────┼──────────────────────────────────────────┤
│ table.raw()        │ [["field", "value"], ["field2", "value2"]]│
│ table.rows()       │ Same as raw() but skips header row       │
│ table.rowsHash()   │ {"field": "value", "field2": "value2"}  │
│ table.hashes()     │ [{col1: "v1", col2: "v2"}, ...]        │
│                    │ (each row as object with header keys)    │
└────────────────────┴──────────────────────────────────────────┘
```

**How It Works:** When the last argument of a Gherkin step is a table, Cucumber passes a `DataTable` object as the final parameter of your function -- after any `{string}`/`{int}` captures. The method you call decides the shape: `raw()` for key-value pairs (loop over `[field, value]`), `hashes()` for multi-column tables where the first row is treated as headers and each following row becomes an object. The profile-form step delegates each row to `getByLabel(field).fill(value)`, which is the same label-first resolution the built-in `When I fill in "Label" with "value"` step uses -- the table is just a loop around the behavior testers already know. The verify step shows the matching read-side pattern: scope a `tr` by the name column, then assert the role text inside that row only.

**At scale:** Tables are the cheapest way to keep product owners inside the feature files. At Vardot, the registration scenarios that survived years of Varbase releases were the table-driven ones -- when a field was added, a BA edited one table row in a pull request and never touched JavaScript. Also remember webship-js already ships `When I fill in the following:` as a built-in; write a custom table step only when you need behavior the built-in does not cover, like custom widgets or per-field validation checks.

---

## Recipe 168: Steps with Doc Strings

**Problem:** You are testing a rich-text editor, or posting a JSON payload, and the content spans multiple paragraphs with quotes, apostrophes, and angle brackets. Squeezing that into a single quoted `{string}` parameter produces an unreadable escaping nightmare -- and the first reviewer who tries to tweak the test data breaks the quoting and the whole feature file stops parsing.

**Solution:**

```javascript
const { When, Then } = require('@cucumber/cucumber');

When('I fill in the editor with:', async function (content) {
  // content is the text between the triple quotes
  await this.page.locator('.rich-editor').fill(content);
});

When('I submit the API request with body:', async function (jsonString) {
  const body = JSON.parse(jsonString);
  // Use body for API request...
});
```

```gherkin
Feature: Content Editor
  Scenario: Write a long blog post
    When I fill in the editor with:
      """
      This is a long blog post about automated testing.

      It spans multiple paragraphs and includes special characters:
      quotes "like these", apostrophes it's, and <html> tags.
      """

  Scenario: Submit JSON payload
    When I submit the API request with body:
      """
      {
        "title": "New Article",
        "body": "Article content here",
        "tags": ["testing", "automation"]
      }
      """
```

**How It Works:** Everything between the `"""` fences arrives in your function as one plain string, last parameter, with the common leading indentation stripped -- so the JSON example reaches `JSON.parse()` exactly as written, no escaping needed. Nothing inside a doc string is interpreted by Gherkin: quotes, `<html>` tags, and blank lines all pass through untouched. This is the same mechanism behind the built-in API steps (`When I send a POST request to "/path" with body:`) and the network-mock steps (`Given the URL "**/api/x" returns the JSON:`) -- check those before writing your own, because for API payloads and mocked responses the built-ins already parse, send, and assert for you. Reserve custom doc-string steps for content that goes *into the UI*, like the rich-editor example.

---

## Recipe 169: Domain-Specific Steps

**Problem:** Here is a scene every QA lead knows: a product owner writes "when the customer places an order, inventory decreases", QA translates it into eleven lines of clicks and fills, and three sprints later nobody -- not even the author -- can tell which business rule those eleven lines were protecting. The PO stops reading the feature files, sign-off becomes "I trust you", and the suite quietly drifts away from the acceptance criteria it was supposed to encode.

**Solution:**

```javascript
// tests/step-definitions/ecommerce-domain.js
const { Given, When, Then } = require('@cucumber/cucumber');
const assert = require('assert');

// Domain: E-commerce
Given('the product {string} is in stock', async function (product) {
  // Navigate and verify
  await this.page.goto(this.launchUrl + '/products');
  await this.page.waitForLoadState('networkidle');
  const row = this.page.locator('tr').filter({ hasText: product });
  const status = await row.locator('.stock-status').textContent();
  assert(status.includes('In Stock'), `${product} is not in stock`);
});

When('the customer places an order for {string}', async function (product) {
  await this.page.goto(this.launchUrl + '/products');
  await this.page.locator('.product-card').filter({ hasText: product })
    .locator('button', { hasText: 'Add to Cart' }).click();
  await this.page.goto(this.launchUrl + '/checkout');
  await this.page.getByRole('button', { name: 'Place Order' }).click();
  await this.page.waitForLoadState('networkidle');
});

Then('the order should be confirmed', async function () {
  await this.page.locator('body').filter({ hasText: 'Order Confirmed' })
    .waitFor({ timeout: 10000 });
});

Then('the inventory for {string} should decrease by {int}', async function (product, amount) {
  // Check inventory via API or admin panel
  await this.page.goto(this.launchUrl + '/admin/inventory');
  await this.page.waitForLoadState('networkidle');
  // Verify inventory count...
});
```

```gherkin
Feature: Order Processing
  As a customer
  I want to order products
  So that I can receive them at home

  Scenario: Successful order reduces inventory
    Given the product "Wireless Mouse" is in stock
    When the customer places an order for "Wireless Mouse"
    Then the order should be confirmed
    And the inventory for "Wireless Mouse" should decrease by 1
```

**Imperative vs Declarative comparison:**

```
Imperative (technical):           Declarative (business):
Given I am on homepage            Given the product "Mouse" is
When I go to "/products"            in stock
And I click "Add to Cart"         When the customer places an
  in the "Mouse" row                order for "Mouse"
And I go to "/checkout"           Then the order should be
And I fill in "Name" with "J"       confirmed
And I fill in "Address" with "1"
And I press "Place Order"
And wait for AJAX to finish
Then I should see "Confirmed"
```

Both test the same thing, but the declarative version communicates the **intent** while the imperative version communicates the **mechanism**.

**How It Works:** Each domain step packages a whole user journey behind one business sentence. `the customer places an order for {string}` navigates, scopes the product card by text, clicks Add to Cart, and submits checkout -- five UI operations, one Gherkin line. The win is not just readability: when checkout gains a confirmation modal next quarter, you update one function instead of every scenario that orders something. The confirmation assertion uses `waitFor({ timeout: 10000 })` deliberately -- order processing crosses a payment round-trip, so it deserves more than the default 5s web-first window. Keep the boundary clean: domain steps describe *what the business does*, the function body owns *how the app does it*. The moment a selector leaks into your Gherkin, you have lost the layer.

**At scale:** This is where multi-project teams win or lose. Define the domain vocabulary once per product domain -- "places an order", "publishes an article", "approves the request" -- and reuse the same phrasing across every project in that domain, even when the implementations differ per site. Product owners then read any project's report and understand it, the PMO can compare quality signals across products because scenarios share a language, and a QA engineer moving between projects already knows the steps. The feature files become the contract; the step definitions absorb the per-project differences.

---

## Recipe 170: Flexible Step Patterns

**Problem:** Five people write feature files on your team, and each phrases the same action a little differently: "I add 1 item", "I add 3 items", "the page should show", "the screen should display". You can either police the wording in every review -- and watch undefined-step errors pile up anyway -- or write patterns tolerant enough to accept the natural variations people actually type.

**Solution:**

```javascript
// Regex with optional parts
When(/^I (add|remove) (\d+) items? (to|from) the cart$/, async function (action, count, preposition) {
  // Matches:
  //   "I add 1 item to the cart"
  //   "I add 3 items to the cart"
  //   "I remove 1 item from the cart"
  //   "I remove 5 items from the cart"
});

// Optional pronoun
When(/^(?:I |we )?click the submit button$/, async function () {
  // Matches: "I click the submit button" AND "click the submit button"
});

// Alternative words
Then(/^the (page|screen|view) should (show|display|contain) "([^"]*)"$/, async function (container, verb, text) {
  // Matches: "the page should show 'Hello'"
  //          "the screen should display 'Hello'"
  //          "the view should contain 'Hello'"
});
```

**How It Works:** This is the territory where regex earns its keep over Cucumber Expressions. `items?` makes the trailing "s" optional, so singular and plural both match. `(?:I |we )?` is a *non-capturing* optional group -- the `?:` prefix means it does not produce a function parameter, which keeps your argument list aligned with the captures you actually care about. Alternations like `(add|remove)` capture which word matched, so one function can branch on the action instead of two near-identical definitions drifting apart. Always anchor with `^` and `$` -- an unanchored pattern can match in the middle of a longer step and collide with another definition, and Cucumber's ambiguous-step error on a Friday afternoon is nobody's favorite.

**Pro Tip:** Flexibility has a budget. Two or three tolerant patterns per domain keep authors happy; a regex with six alternations becomes a step nobody can discover or document. webship-js's own rule is a good one to copy: every step gets at least five documented example lines, so the accepted phrasings are written down where authors can find them.

**At scale:** Across many projects, prefer *one* published phrasing per action and use flexible patterns only as a compatibility net for the variants that already exist in older suites. A shared step-reference page (or the built-in step inventory) beats clever regex -- new hires learn the canonical wording once and carry it to every project they touch.

---

## Step Definition Organization

```
tests/
├── step-definitions/
│   ├── webship.js           ← Built-in (from webship-js)
│   ├── webship-api.js       ← Built-in API (from webship-js)
│   ├── custom-auth.js       ← Your login/logout steps
│   ├── custom-cart.js       ← Your e-commerce steps
│   ├── custom-admin.js      ← Your admin panel steps
│   ├── custom-viewport.js   ← Your responsive testing steps
│   └── custom-a11y.js       ← Your accessibility steps
├── features/
│   ├── auth/
│   │   ├── login.feature
│   │   └── registration.feature
│   ├── products/
│   │   ├── listing.feature
│   │   └── cart.feature
│   └── admin/
│       ├── dashboard.feature
│       └── users.feature
└── support/
    └── hooks.js             ← Custom hooks
```

---

## Chapter Summary

- Step definitions connect Gherkin text to Playwright automation code
- Three parts: keyword (Given/When/Then), pattern (expression/regex), function (async code)
- Cucumber Expressions are simpler; Regular Expressions are more flexible
- `{string}` captures quoted text, `{int}` captures numbers
- Data tables pass structured key-value or multi-column data
- Doc strings pass multiline text (JSON, HTML, long content)
- Domain-specific steps make Gherkin read like business language
- Flexible patterns handle pluralization and wording variations
- Organize step definitions by domain (auth, cart, admin) not by keyword
- All steps share the same World (`this`) within a scenario

Next: Hooks and Lifecycle -- controlling what happens before, during, and after your tests.

# Chapter 28: Building a Complete Test Suite

---

This chapter brings everything together. We will build a complete test suite for a realistic e-commerce application, from project setup to CI configuration. Every recipe in this book is a building block. Now we assemble them.

One framing before we start: we are not just writing tests, we are building an asset the whole team operates. The measure of a complete suite is not coverage -- it is whether a new hire can clone it, run it, and read last night's report on their first morning, and whether the same structure can be stamped onto the next project without a redesign.

---

## The Application: ShopDemo

```
┌──────────────────────────────────────────────────────────────────┐
│ ShopDemo - E-Commerce Application                                │
│                                                                  │
│  Public Pages:                     Admin Panel:                  │
│  ├── Homepage                      ├── Dashboard                │
│  ├── Product Catalog               ├── Product Management       │
│  ├── Product Detail                ├── Order Management         │
│  ├── Shopping Cart                 ├── User Management          │
│  ├── Checkout                      ├── Content/Pages            │
│  ├── Order Confirmation            └── Settings                 │
│  ├── User Registration                                          │
│  ├── Login / Logout               API Endpoints:                │
│  ├── User Profile                  ├── GET/POST /api/products   │
│  └── Search                        ├── GET/POST /api/orders     │
│                                    ├── GET/POST /api/users      │
│  User Roles:                       ├── POST /api/auth/login     │
│  ├── Guest (not logged in)         └── GET /api/search          │
│  ├── Customer (registered)                                      │
│  ├── Editor (content management)                                │
│  └── Admin (full access)                                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

ShopDemo is deliberately shaped to exercise every pattern in this book: form-heavy auth flows, a stateful cart, admin CRUD behind role-gated routes, a JSON API, and public pages that have to clear an accessibility bar. Four roles give us a permission matrix; the API gives us a fast non-browser tier. If your real application is a CMS or a SaaS dashboard instead of a shop, the shapes still map one-to-one -- swap "product" for whatever your domain calls a record.

---

## Complete Project Structure

```
shopdemo-tests/
├── package.json
├── cucumber.js                      ← Configuration profiles
├── playwright.config.ts             ← Browser config
│
├── tests/
│   ├── features/
│   │   ├── auth/
│   │   │   ├── login.feature
│   │   │   ├── registration.feature
│   │   │   └── logout.feature
│   │   ├── products/
│   │   │   ├── catalog.feature
│   │   │   ├── search.feature
│   │   │   └── product-detail.feature
│   │   ├── cart/
│   │   │   ├── add-to-cart.feature
│   │   │   └── cart-management.feature
│   │   ├── checkout/
│   │   │   └── checkout-flow.feature
│   │   ├── admin/
│   │   │   ├── dashboard.feature
│   │   │   ├── product-admin.feature
│   │   │   ├── order-admin.feature
│   │   │   └── user-admin.feature
│   │   ├── api/
│   │   │   ├── products-api.feature
│   │   │   ├── orders-api.feature
│   │   │   └── auth-api.feature
│   │   └── accessibility/
│   │       └── a11y-audit.feature
│   │
│   ├── step-definitions/
│   │   ├── auth-steps.js
│   │   ├── cart-steps.js
│   │   ├── viewport-steps.js
│   │   ├── screenshot-steps.js
│   │   └── a11y-steps.js
│   │
│   ├── support/
│   │   ├── hooks.js
│   │   ├── helpers.js
│   │   └── builders.js
│   │
│   └── assets/
│       └── test-image.png
│
├── screenshots/                     ← Auto-generated
├── reports/                         ← Auto-generated
└── .github/workflows/test.yml       ← CI configuration
```

The layout works with webship-js's defaults instead of against them: Cucumber discovers step definitions from `tests/step-definitions/**/*.js` and features from `tests/features/**/*.feature`, so the structure is convention, not configuration. Features are foldered by business domain -- auth, cart, checkout -- never by test type, because when checkout breaks you want every checkout scenario in one place, whether it drives a browser or the API. Custom step files stay thin; webship-js's 411 built-in steps carry most scenarios, and `support/builders.js` is where test-data construction lives -- which will matter shortly. When selector complexity grows, add a `tests/selectors/` folder with named-selector JSON files so features speak in names and markup churn gets absorbed in one file instead of fifty.

**At scale:** This tree is your template repository. New project: clone it, change `LAUNCH_URL`, delete the features that do not apply, and the new suite inherits the structure, the CI workflow, and the conventions on day one. Twenty projects with the same skeleton means any QA engineer can navigate any suite. The alternative -- twenty hand-grown layouts -- is how teams end up with suites only their original author can maintain.

---

## The Feature Files

### Authentication

Login goes first and gets the `@smoke` tag, because it is the cheapest possible canary: if login is down, nothing else matters and half the suite cannot even start.

```gherkin
# tests/features/auth/login.feature
@smoke @auth
Feature: User Login
  As a registered user
  I want to log in
  So that I can access my account

  Scenario: Successful login
    Given I am on homepage
    When I go to "/login"
    And I fill in "Email" with "customer@example.com"
    And I fill in "Password" with "CustPass123"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "My Account"
    And I should not see "Sign In"

  Scenario: Failed login
    Given I am on homepage
    When I go to "/login"
    And I fill in "Email" with "customer@example.com"
    And I fill in "Password" with "WrongPassword"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "Invalid credentials"
    And I should see "Sign In"

  Scenario: Empty credentials
    Given I am on homepage
    When I go to "/login"
    And I press "Sign In"
    Then I should see "Email is required"
```

Look at what the steps are quietly doing for you. `I fill in "Email"` resolves the field label-first -- it finds the label with that text and follows it to the input, falling back to id, name, and placeholder -- so the scenario survives the markup refactor that renames `#edit-email` to `#login-email`. After `I press "Sign In"`, webship-js's settle hook waits for the login request to drain and the DOM to go quiet before the next step fires, and the explicit `wait for AJAX to finish` adds a checkpoint a reader can see. The three scenarios are the three things that actually happen at a login form: right password, wrong password, empty submit. And the negative assertion in the first scenario matters -- `I should not see "Sign In"` proves the page genuinely changed state, which the positive assertion alone cannot.

**Pro Tip:** This feature tests login. The rest of the suite just needs to *be* logged in, which is a different requirement. webship-js ships auth-state steps that save, restore, and clear the browser's storage state: drive the form once, save the state, and let every later scenario restore it instead of re-typing credentials (Chapter 10 covers the pattern). With thirty scenarios needing a customer session, that is minutes off every single run.

### Shopping Cart

```gherkin
# tests/features/cart/add-to-cart.feature
@smoke @cart
Feature: Shopping Cart
  As a customer
  I want to add products to my cart
  So that I can purchase them

  Scenario: Add a product to cart
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    And I click "Laptop Pro"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"
    And I should see "$999.99"
    When I press "Add to Cart"
    And wait for AJAX to finish
    Then I should see "Added to cart"

  Scenario: View cart with items
    Given I am a logged in user with the "customer" user
    When I go to "/products"
    And wait for AJAX to finish
    And I click "Laptop Pro"
    And wait for AJAX to finish
    And I press "Add to Cart"
    And wait for AJAX to finish
    When I go to "/cart"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"
    And I should see "$999.99"
    And I should see "Quantity"

  Scenario: Remove item from cart
    Given I am a logged in user with the "customer" user
    When I go to "/cart"
    And wait for AJAX to finish
    And I click "Remove" in the "Laptop Pro" row
    And wait for AJAX to finish
    Then I should not see "Laptop Pro"
    And I should see "Your cart is empty"
```

The cart is where state starts to bite. The first two scenarios build their own state -- they add the laptop before asserting on it -- but the third does not: it goes straight to `/cart` and expects "Laptop Pro" to already be there, which is only true because the previous scenario left it behind for the same logged-in customer. That coupling works for months and then explodes the day you add `--parallel`, when scenarios stop running in file order. In a real suite, give the removal scenario its own setup -- repeat the add steps, or seed the cart through the API tier -- so every scenario owns its state. Meanwhile the row-scoped click is doing quiet, important work: with three products in the cart there are three "Remove" links, and `in the "Laptop Pro" row` is what keeps the step from grabbing the first one it finds.

**At scale:** Shared test accounts are the multi-suite version of the same trap. When four projects' nightly runs all log in as `customer@example.com` against one shared staging backend, one suite's leftover cart fails another suite's empty-cart assertion -- and the report blames the wrong project. Give each suite, and ideally each parallel worker, its own user, and write the account-naming convention into your shared template so nobody rediscovers the collision at 2 a.m.

### Admin Product Management

```gherkin
# tests/features/admin/product-admin.feature
@regression @admin
Feature: Product Management
  As an admin
  I want to manage the product catalog
  So that customers see accurate product information

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/products"
    And wait for AJAX to finish

  Scenario: View product list
    Then I should see "Products"
    And I should see "Laptop Pro"
    And I should see "Wireless Mouse"

  Scenario: Create a new product
    When I press "Add Product"
    And wait for AJAX to finish
    And I fill in "Name" with "USB-C Hub"
    And I fill in "Price" with "49.99"
    And I fill in "SKU" with "HUB-001"
    And I select "Accessories" from "Category"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Product created"
    When I go to "/admin/products"
    And wait for AJAX to finish
    Then I should see "USB-C Hub"

  Scenario: Edit a product
    When I click "Edit" in the "Laptop Pro" row
    And wait for AJAX to finish
    And I fill in "Price" with "899.99"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Product updated"

  Scenario: Delete a product
    When I click "Delete" in the "USB-C Hub" row
    And wait for the modal to appear
    And I click "Confirm" in the modal
    And wait for AJAX to finish
    Then I should see "Product deleted"
    And I should not see "USB-C Hub"
```

`Background` runs before every scenario in the feature -- log in, navigate, settle -- which saves three lines per scenario and, more importantly, makes every scenario start from the same known place. Read the CRUD lifecycle in order, though: create the hub, edit the laptop, delete the hub. That delete depends on the create two scenarios earlier -- the same ordering coupling the cart feature has, tolerable in a demo and worth fixing in production (create-and-delete inside one scenario, or seed through the API tier). The modal steps earn their keep on the destructive action: `wait for the modal to appear` polls for the dialog instead of assuming it opened instantly, and `I click "Confirm" in the modal` scopes the click so it cannot hit some other "Confirm" elsewhere on the page.

### API Tests

```gherkin
# tests/features/api/products-api.feature
@regression @api
Feature: Products API
  Background:
    Given the API base URL is "http://localhost:8080"
    Given I am authenticating as "admin" with "AdminPass123" password

  Scenario: List all products
    When I send a GET request to "/api/products"
    Then the API response code should be 200
    And the response should be valid JSON
    And the API response should contain "Laptop Pro"

  Scenario: Create a product via API
    When I send a POST request to "/api/products" with body:
      """
      {
        "name": "Test Product",
        "price": 19.99,
        "sku": "TEST-001",
        "category": "Test"
      }
      """
    Then the API response code should be 201
    And the JSON response should have "name" equal to "Test Product"
    And the JSON response should have property "id"

  Scenario: Get a single product
    When I send a GET request to "/api/products/1"
    Then the API response code should be 200
    And the JSON response should have property "name"
    And the JSON response should have property "price"

  Scenario: Unauthenticated API access fails
    When I send a GET request to "/api/admin/users"
    Then the API response code should be 200
    # Admin endpoints require admin auth, which we have
```

These scenarios never launch a browser. The API steps drive HTTP requests directly through axios, which is why this whole feature finishes in seconds and makes a perfect early pipeline stage: if `POST /api/products` is broken, you want to know before spending browser minutes watching the admin form fail for the same reason. The `Background` applies to every scenario in the feature -- and that is exactly the trap in the last one. Its name says unauthenticated access fails, but the Background has already authenticated as admin, so 200 is all it can ever see; a genuine unauthenticated check has to live in a feature file without that Background. Scenario names are documentation. When the name and the assertion disagree, the next reader inherits the confusion -- treat that mismatch as a bug in the test, even when the run is green.

### Accessibility

```gherkin
# tests/features/accessibility/a11y-audit.feature
@a11y
Feature: Accessibility
  Scenario: Homepage is accessible
    Given I am on homepage
    And wait for the page to load
    Then the page should have no critical accessibility violations

  Scenario: Login page is accessible
    Given I am on homepage
    When I go to "/login"
    And wait for the page to load
    Then the page should have no critical accessibility violations

  Scenario: Product listing is accessible
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    And wait for the page to load
    Then the page should have no critical accessibility violations
```

The audit step runs the bundled axe-core engine against the live, rendered page -- computed styles, injected widgets and all -- and fails only on findings axe rates critical. That is the right opening bar for an application that has never been audited; once the criticals are gone, the stricter sibling `Then the page should pass an accessibility audit at level "AA"` applies the full WCAG AA rule set, and `Then print accessibility violations` dumps each finding with its impact and help URL into the report when you need the detail. Keep the `@a11y` tag on these scenarios: audits cost more time than functional checks and belong in the nightly tier, not in every pull-request run.

---

## The Configuration

```javascript
// cucumber.js
const common = {
  require: [
    'node_modules/webship-js/tests/step-definitions/**/*.js',
    'tests/step-definitions/**/*.js',
    'tests/support/**/*.js',
  ],
  format: ['@cucumber/pretty-formatter'],
};

module.exports = {
  default: {
    ...common,
    worldParameters: {
      launchUrl: 'http://localhost:8080',
      minWaitTime: {
        before_scenario: 0, after_scenario: 0,
        before_step: 0, after_step: 0, page: 3000,
      },
    },
  },
  ci: {
    ...common,
    format: ['@cucumber/pretty-formatter', 'json:reports/report.json'],
    worldParameters: {
      launchUrl: process.env.LAUNCH_URL || 'http://localhost:8080',
      minWaitTime: {
        before_scenario: 0, after_scenario: 0,
        before_step: 0, after_step: 0, page: 3000,
      },
    },
  },
};
```

Two profiles, one shared `common` block. The `require` array is what wires webship-js in: the first path loads the framework's 411 built-in steps out of `node_modules`, the next two load your custom steps and hooks -- delete that first line and every built-in step suddenly "does not exist". `minWaitTime.page: 3000` is the per-navigation settle budget: after each page load, webship-js gives the application up to three seconds to finish its AJAX, drain its timers, and stop mutating the DOM before the next step runs. The `ci` profile changes exactly two things: it adds a JSON formatter -- the input that `bin/generate-reports.js` later turns into HTML and PDF reports -- and it reads `LAUNCH_URL` from the environment instead of hard-coding localhost. That second change is the important one.

**At scale:** `LAUNCH_URL` as an environment variable is the difference between one suite that tests every environment and four diverging copies that each test one. The pipeline sets `LAUNCH_URL=https://staging.example.com` for the staging job and the production URL for the post-deploy smoke job -- same repo, same features, zero edits. Apply the rule everywhere: base URLs and secrets live in CI variables, never in committed config. Across twenty projects, that single convention is what lets a platform engineer rewire an environment without opening twenty repos.

---

## Running the Suite

```bash
# Smoke tests (fast feedback)
npx cucumber-js --tags "@smoke"

# Full regression
npx cucumber-js --tags "not @wip"

# API tests only (no browser needed)
npx cucumber-js --tags "@api"

# Accessibility audit
npx cucumber-js --tags "@a11y"

# Full suite, all browsers, parallel
BROWSER=chromium npx cucumber-js --parallel 4 --profile ci
```

The tag tiers map straight onto pipeline stages: `@smoke` on every push for feedback in minutes, the full regression before a deploy, `@api` as the cheap early gate that needs no browser at all, `@a11y` on the nightly schedule. `--parallel 4` starts four Cucumber workers, each driving its own browser -- which roughly quarters the wall time and doubles as an honesty test: any scenario leaning on another scenario's leftovers (remember the cart) starts failing intermittently the day you switch it on. Better to find that out now than during release week.

**At scale:** The JSON report is the half of this output your managers actually consume -- once `bin/generate-reports.js` turns it into HTML with metadata stamped in: `--app-version 2.3.1 --env staging --browser chromium`. Use identical metadata keys in every project's pipeline, because that is what makes a cross-project dashboard possible. And that dashboard is the real answer to "is release X safe?" -- a green run with a version number on it that a delivery manager can read unaided, instead of a QA engineer's recollection relayed in a status meeting.

---

## Chapter Summary

- A complete test suite covers: auth, CRUD, cart, checkout, API, admin, accessibility
- Organize by business domain, not by test type
- Use tags for tiered execution: @smoke, @regression, @api, @a11y
- Background steps reduce repetition within a feature
- API tests complement browser tests (faster, more precise)
- CI configuration ties everything together

Next: BDD in the Age of AI -- how artificial intelligence transforms testing.

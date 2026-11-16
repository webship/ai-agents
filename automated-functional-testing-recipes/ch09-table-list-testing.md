# Chapter 9: Table and List Testing

---

Tables are the backbone of data-driven web applications. Admin panels, dashboards, reports, user management screens, order lists, inventory systems -- all of them present data in tabular form. If your application has a backend, it has tables.

Testing tables introduces a unique challenge: you are not just checking that text exists on the page, you need to verify that the right data appears in the right row. The product name "Laptop" should appear in the same row as the price "$999" and the status "In Stock." A simple `I should see "Laptop"` tells you the text is somewhere on the page but not whether it is in the correct context.

This chapter gives you recipes for every table interaction pattern: reading data, clicking actions within rows, sorting, filtering, pagination, bulk operations, and more.

---

## Understanding Table Structure

Before we dive into recipes, let us understand the HTML structure that webship-js targets:

```
HTML Table Structure:
┌─────────────────────────────────────────────────────────────────┐
│ <table>                                                         │
│   <thead>                                                       │
│     <tr>                                   ← Header Row        │
│       <th>Name</th>                                             │
│       <th>Email</th>                                            │
│       <th>Role</th>                                             │
│       <th>Actions</th>                                          │
│     </tr>                                                       │
│   </thead>                                                      │
│   <tbody>                                                       │
│     <tr>                                   ← Data Row 1        │
│       <td>John Smith</td>                                       │
│       <td>john@example.com</td>                                 │
│       <td>Admin</td>                                            │
│       <td><a>Edit</a> <button>Delete</button></td>             │
│     </tr>                                                       │
│     <tr>                                   ← Data Row 2        │
│       <td>Jane Doe</td>                                         │
│       <td>jane@example.com</td>                                 │
│       <td>Editor</td>                                           │
│       <td><a>Edit</a> <button>Delete</button></td>             │
│     </tr>                                                       │
│   </tbody>                                                      │
│ </table>                                                        │
└─────────────────────────────────────────────────────────────────┘
```

webship-js row-based steps work by:
1. Finding all `<tr>` elements on the page
2. Filtering to the row that contains the **row identifier** text
3. Performing the action within that specific row

This means you can interact with any cell in a row by identifying the row with any unique text it contains (name, email, ID, etc.).

---

## Recipe 89: Verifying Table Content

**Problem:** Your admin user list shows the right names and the right roles -- until the day it does not. A permissions refactor swaps two columns in the API response, and suddenly every user displays someone else's role. Your old test passed anyway, because `I should see "Admin"` only proves the word "Admin" exists *somewhere* on the page. What you actually need to prove is that "Admin" sits in the same row as john@example.com -- and that is a different assertion entirely.

**Solution:**

```gherkin
Feature: Table Content Verification
  As an admin
  I want to see user data in a table
  So that I can manage the system

  Scenario: Table shows expected data
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And wait for AJAX to finish
    Then I should see "John Smith"
    And I should see "john@example.com"
    And I should see "Jane Doe"
    And I should see "jane@example.com"

  Scenario: Verify data in specific rows
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And wait for AJAX to finish
    Then I should see "Admin" in the "john@example.com" row
    And I should see "Editor" in the "jane@example.com" row
    And I should see "Active" in the "john@example.com" row
    And I should see "Active" in the "jane@example.com" row

  Scenario: Verify data NOT in a row
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And wait for AJAX to finish
    Then I should not see "Blocked" in the "john@example.com" row
    And I should not see "Admin" in the "jane@example.com" row
```

**How It Works:**

`Then I should see "Admin" in the "john@example.com" row` does the following:

```javascript
// What webship-js does internally
const rows = Array.from(document.querySelectorAll('tr'));
const found = rows
  .filter(r => r.textContent.includes('john@example.com'))  // Find the row
  .some(r => r.textContent.includes('Admin'));               // Check for text
```

```
Row Matching:
┌──────────────┬──────────────────┬────────┬─────────┐
│ Name         │ Email            │ Role   │ Actions │
├──────────────┼──────────────────┼────────┼─────────┤
│ John Smith   │ john@example.com │ Admin  │ Edit    │ ← This row matches
│ Jane Doe     │ jane@example.com │ Editor │ Edit    │   "john@example.com"
└──────────────┴──────────────────┴────────┴─────────┘
                        │                      │
              Row identifier             Value to check
```

The two-text approach is what makes this assertion meaningful. The row identifier narrows the search space to one `<tr>`; the value check then runs only inside that row's text content. If "Admin" appears in Jane's row but not John's, the step fails -- exactly the column-swap bug a page-wide `should see` would have missed. Note also the `wait for AJAX to finish` before the assertions: admin tables almost always load their rows through a fetch after the page shell renders, and webship-js tracks those in-flight requests through its `window.__webshipAjaxCount` wrapper, so the assertion only runs once the data has actually landed in the DOM.

**Choosing good row identifiers:**

The row identifier should be a value that uniquely identifies a single row in the table. Good choices:
- Email addresses (unique by nature)
- IDs or reference numbers (`ORD-12345`)
- Usernames
- Full names (if unique in the test data)

Bad choices:
- Status values ("Active" -- multiple rows might have this)
- Common text ("Edit" -- every row has an Edit button)

**At scale:** When you run this pattern across many projects, the failure mode is not the step -- it is the test data. One project seeds john@example.com, another seeds admin@client-a.test, and your scenarios stop being portable. Standardize the seed fixtures: the same set of test users with the same emails and roles in every project, loaded by the same seed script. Then the same `in the "..." row` scenarios can be copied between suites verbatim, and a new QA engineer can read any project's table tests without asking who "Bob Wilson" is supposed to be.

---

## Recipe 90: Clicking Actions in Table Rows

**Problem:** Every row in your user table has an Edit link and a Delete button -- fifty rows, fifty identical "Edit" labels. A plain `When I click "Edit"` grabs the first one it finds, which is fine until a new user is seeded above your target row and the test quietly starts editing the wrong account. The day someone notices, the test has been "passing" for three weeks while validating nothing. You need the click scoped to a specific row, not to whichever row happens to be first.

**Solution:**

```gherkin
Feature: Table Row Actions
  Background:
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And wait for AJAX to finish

  Scenario: Edit a user from the table
    When I click "Edit" in the "john@example.com" row
    And wait for AJAX to finish
    Then I should see "Edit User"
    And I should see "John Smith"

  Scenario: Delete a user from the table
    When I click "Delete" in the "jane@example.com" row
    And wait for the modal to appear
    Then I should see "Are you sure?" in the modal
    When I click "Confirm" in the modal
    And wait for AJAX to finish
    Then I should not see "jane@example.com"

  Scenario: View details from the table
    When I click "View" in the "john@example.com" row
    And wait for AJAX to finish
    Then I should see "User Details"
    And I should see "John Smith"
    And I should see "Admin"

  Scenario: Toggle status from the table
    Then I should see "Active" in the "john@example.com" row
    When I click "Deactivate" in the "john@example.com" row
    And wait for AJAX to finish
    Then I should see "Inactive" in the "john@example.com" row
```

**How It Works:**

`When I click "Edit" in the "john@example.com" row` does the following:

1. Finds the `<tr>` that contains "john@example.com"
2. Within that row, searches for `<a>`, `<button>`, `[role="button"]`, `.btn`, or `<input type="submit">` elements
3. Filters to the one with text matching "Edit"
4. Clicks it

```
Search Strategy:
┌───────────────────────────────────────────────────────────────┐
│ <tr>                                                          │
│   <td>John Smith</td>                                         │
│   <td>john@example.com</td>   ← Row found by this text      │
│   <td>Admin</td>                                              │
│   <td>                                                        │
│     <a href="/edit/1">Edit</a>       ← Clickable match!      │
│     <button>Delete</button>          ← Not this one          │
│   </td>                                                       │
│ </tr>                                                         │
└───────────────────────────────────────────────────────────────┘

Priority: a > button > [role="button"] > .btn > input[type="submit"]
If no clickable element matches, falls back to getByText exact match.
```

The priority order matters more than it looks. Action cells often contain both a visible "Edit" link and a hidden form button with similar text; searching anchors first means the step clicks what a human would click. And because the search is scoped to the matched `<tr>`, the other forty-nine "Edit" links on the page are never candidates -- the wrong-row bug from the Problem statement becomes structurally impossible.

After the click, the AfterStep hook runs `smartSettle(page, 1500)` automatically: it waits for in-flight fetch/XHR calls to drain and the DOM to go quiet before your next step runs. That is why "click Delete, confirm in modal, assert row gone" works without any sleep between steps.

**At scale:** Across a portfolio of admin UIs, the action labels drift -- one project says "Edit", another "Modify", a third uses an icon-only button with an aria-label. Resist writing per-project step variants. Instead, push back on the products: a shared convention ("row actions are anchors or buttons with visible text or aria-labels from an agreed vocabulary") costs the developers nothing and lets one set of scenarios cover every project. This is a conversation a QA lead should have once, not a workaround each suite carries forever.

---

## Recipe 91: Testing Table with CRUD Operations

**Problem:** Create works. Edit works. Delete works. You have tested each one -- in isolation, against pre-seeded data. Then a release ships where creating a product succeeds but the new row never appears in the list, because the list query caches aggressively. No isolated test catches that; only the full lifecycle does: create it, see it in the table, change it, see the change, delete it, see it gone. This is the test that fails when the pieces stop talking to each other.

**Solution:**

```gherkin
Feature: Table CRUD Operations
  As an admin managing products
  I want to create, view, edit, and delete products
  So that I can maintain the product catalog

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/products"
    And wait for AJAX to finish

  Scenario: Create a new product and see it in the table
    When I press "Add Product"
    And wait for AJAX to finish
    And I fill in "Name" with "Wireless Keyboard"
    And I fill in "Price" with "49.99"
    And I fill in "SKU" with "KB-100"
    And I select "Accessories" from "Category"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Product created"

    When I go to "/admin/products"
    And wait for AJAX to finish
    Then I should see "Wireless Keyboard"
    And I should see "$49.99" in the "Wireless Keyboard" row
    And I should see "KB-100" in the "Wireless Keyboard" row
    And I should see "Accessories" in the "Wireless Keyboard" row

  Scenario: Edit a product from the table
    When I click "Edit" in the "Wireless Keyboard" row
    And wait for AJAX to finish
    And I fill in "Price" with "39.99"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Product updated"

    When I go to "/admin/products"
    And wait for AJAX to finish
    Then I should see "$39.99" in the "Wireless Keyboard" row

  Scenario: Delete a product from the table
    When I click "Delete" in the "Wireless Keyboard" row
    And wait for the modal to appear
    And I click "Confirm" in the modal
    And wait for AJAX to finish
    Then I should see "Product deleted"
    And I should not see "Wireless Keyboard"
```

**How It Works:**

Notice the structure of the create scenario: save the form, check the success message, then navigate back to the list with `I go to "/admin/products"` and verify the row there. The re-navigation is deliberate. Many apps show an optimistic success message before the database write commits, or render the list from a cache. By forcing a fresh page load, the test reads what the backend actually persisted -- not what the previous page claimed. The `in the "Wireless Keyboard" row` assertions then verify all four fields landed in one row, which catches partial saves (name persisted, category dropped) that a single `should see` misses.

One caution: these three scenarios depend on each other -- edit assumes create ran, delete assumes both did. Cucumber runs scenarios in file order by default, so this works, but it breaks the moment you run scenarios in parallel or filter by tag. For a suite you will maintain long-term, prefer one longer scenario covering the full lifecycle, or give each scenario its own setup.

**Pro Tip:** End the lifecycle with delete even when delete is not what you are testing. A CRUD scenario that cleans up after itself can run twice in a row without "Wireless Keyboard already exists" failures -- which is exactly what happens when a CI retry re-runs the feature.

---

## Recipe 92: Testing Sortable Tables

**Problem:** A customer reports that sorting the product list by price puts a $9.99 cable between two $900 laptops -- the column sorts alphabetically because the backend treats price as a string. Nobody caught it because nobody tests sorting; it is the kind of feature that "obviously works" until it does not. You want a scenario that clicks each sortable header and proves the order actually changed, without turning into a brittle position-by-position comparison.

**Solution:**

```gherkin
Feature: Sortable Table
  As a user viewing a list of products
  I want to sort by different columns
  So that I can find what I need quickly

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/products"
    And wait for AJAX to finish

  Scenario: Sort by name ascending
    When I click "Name"
    And wait for AJAX to finish
    Then I should see "Apple Watch"
    # First alphabetical item should be visible at the top

  Scenario: Sort by name descending
    When I click "Name"
    And wait for AJAX to finish
    # Click again to reverse sort
    When I click "Name"
    And wait for AJAX to finish
    Then I should see "Wireless Mouse"
    # Last alphabetical item should now be at the top

  Scenario: Sort by price
    When I click "Price"
    And wait for AJAX to finish
    Then I should see "$9.99"
    # Cheapest item should be visible

  Scenario: Sort by date
    When I click "Created"
    And wait for AJAX to finish
    Then I should see "2024-01-01"
    # Oldest item should be visible
```

**How It Works:**

Sortable tables typically use `<th>` elements as click targets. `When I click "Name"` finds the `<a>` or clickable element with text "Name" in the table header and clicks it. The server (or JavaScript) re-orders the rows, and the `wait for AJAX` ensures the new order has loaded. If the sort is server-side, that step watches the fetch/XHR counter drop back to zero; if it is client-side JavaScript re-ordering existing rows, the auto-settle behavior after each step still waits for the DOM mutations to go quiet before the next assertion runs -- so the same scenario works against both implementations without you knowing which one the developers chose.

The assertions lean on a practical trick: with a paginated table, the first page after sorting ascending contains the alphabetically-first item, and after the second click (descending) it contains the last. If "Wireless Mouse" is visible on page one after a descending sort, the sort direction flipped -- and the string-vs-number price bug from the Problem above shows up immediately, because `$9.99` will not be on the first page of a correctly ascending numeric sort.

```
Sort Interaction Flow:
┌─────────────────────────────────────────────┐
│ <thead>                                      │
│   <tr>                                       │
│     <th><a>Name ▲</a></th>  ← Click here    │
│     <th><a>Price</a></th>                    │
│     <th><a>Created</a></th>                  │
│   </tr>                                      │
│ </thead>                                     │
│ <tbody>                                      │
│   <tr><td>Apple Watch</td>...</tr>  ← First │
│   <tr><td>Laptop Pro</td>...</tr>            │
│   <tr><td>Wireless Mouse</td>...</tr>        │
│ </tbody>                                     │
│                                              │
│ After clicking "Name" again (descending):    │
│ <tbody>                                      │
│   <tr><td>Wireless Mouse</td>...</tr>← First│
│   <tr><td>Laptop Pro</td>...</tr>            │
│   <tr><td>Apple Watch</td>...</tr>           │
│ </tbody>                                     │
└─────────────────────────────────────────────┘
```

**Note:** Testing exact sort order is difficult with `should see` because it only checks presence, not position. For most cases, verifying that the expected first/last item is visible after sorting is sufficient. If you need to verify exact order, use Scenario Outlines with expected positions or custom step definitions.

**At scale:** Sort tests only stay portable if your seed data is designed for them. Across multiple projects, seed every list fixture with three sentinel records: one that sorts first alphabetically, one that sorts last, and one with an extreme numeric value. Then the same four sort scenarios work on every project's tables, and a sort regression on any of them fails loudly instead of depending on whatever data a developer happened to have locally.

---

## Recipe 93: Testing Filterable Tables

**Problem:** Support escalates a ticket: an admin filtered the user list by "Blocked" status, exported the result, and emailed warnings to forty accounts -- half of them active, because the filter silently matched nothing and showed everyone. A filter that fails open is worse than no filter, and it is invisible to a test that only checks the rows you expect to see. The day this bites you, you learn that filter tests must assert what disappeared, not just what stayed.

**Solution:**

```gherkin
Feature: Filterable Table
  As an admin viewing users
  I want to filter the table
  So that I can find specific users quickly

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/users"
    And wait for AJAX to finish

  Scenario: Filter by search text
    When I fill in "Search" with "john"
    And wait for AJAX to finish
    Then I should see "John Smith"
    And I should see "john@example.com"
    And I should not see "Jane Doe"
    And I should not see "Bob Wilson"

  Scenario: Filter by role dropdown
    When I select "Admin" from "Role"
    And wait for AJAX to finish
    Then I should see "John Smith"
    And I should not see "Jane Doe"

  Scenario: Filter by status
    When I select "Active" from "Status"
    And wait for AJAX to finish
    Then I should see "John Smith"
    Then I should see "Jane Doe"
    And I should not see "Blocked User"

  Scenario: Combine multiple filters
    When I fill in "Search" with "john"
    And I select "Admin" from "Role"
    And wait for AJAX to finish
    Then I should see "John Smith"
    And I should not see "Jane Doe"

  Scenario: Clear filters shows all data
    When I fill in "Search" with "john"
    And wait for AJAX to finish
    Then I should not see "Jane Doe"

    When I fill in "Search" with ""
    And wait for AJAX to finish
    Then I should see "John Smith"
    And I should see "Jane Doe"

  Scenario: No results found
    When I fill in "Search" with "zzzznonexistent"
    And wait for AJAX to finish
    Then I should see "No results found"
    And I should not see "John Smith"
```

**How It Works:**

```
Filter Flow:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Search: [john          ]  Role: [All ▼]  Status: [All ▼]│
│                                                          │
│  ┌──────────┬──────────────────┬────────┬─────────┐     │
│  │ Name     │ Email            │ Role   │ Status  │     │
│  ├──────────┼──────────────────┼────────┼─────────┤     │
│  │ John     │ john@example.com │ Admin  │ Active  │     │
│  │ Smith    │                  │        │         │     │
│  └──────────┴──────────────────┴────────┴─────────┘     │
│                                                          │
│  Showing 1 of 5 users                                    │
│                                                          │
└──────────────────────────────────────────────────────────┘

The filter step triggers either:
1. Client-side filtering (JavaScript hides non-matching rows)
2. Server-side filtering (AJAX request fetches filtered data)

Either way, "wait for AJAX to finish" handles the timing.
```

`I fill in "Search" with "john"` resolves the field label-first -- it looks for a `<label>` reading "Search", then falls back to id, name, and placeholder attributes. That matters for filters because filter inputs are often placeholder-only ("Search users..."); the step still finds them without a CSS selector. After typing, the smart-wait machinery takes over: whether the app debounces and fires an AJAX request or filters rows with pure JavaScript, the settle check (no pending fetch/XHR, no pending timers, DOM quiet for 250ms) holds the next step until the table has finished re-rendering.

**Tip:** When testing filters, always verify both what SHOULD appear and what SHOULD NOT appear. Just checking "I should see John" does not prove the filter worked -- John might have been visible anyway. The `should not see` assertion for excluded items is what proves the filter is actually filtering. The "fails open" bug in the Problem above passes every positive assertion and fails only the negative one.

**At scale:** The combine-multiple-filters scenario is the one teams skip and the one that breaks most often, because each filter is usually developed and reviewed in isolation. When you maintain table tests across many projects, make "one combined-filter scenario per filterable table" a checklist item in your test plan template. It is one scenario per table, and it is the only automated evidence that filters compose correctly -- the kind of line item a QA lead can point to when a release manager asks what "tested" means.

---

## Recipe 94: Testing Table Pagination

**Problem:** Your order list paginates at ten rows, and for months that was enough -- staging never had more than eight orders. Then production crosses the threshold and the "Next" button 404s, or page two shows the same ten orders as page one because the offset parameter was never wired up. Pagination bugs hide in exactly the data volumes your test environment does not have, which is why they ship. The fix is a seeded dataset big enough to paginate, plus scenarios that walk the boundaries.

**Solution:**

```gherkin
Feature: Table Pagination
  As an admin viewing a long list
  I want to navigate between pages
  So that I can see all records

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/orders"
    And wait for AJAX to finish

  Scenario: First page shows first batch of data
    Then I should see "ORD-001"
    And I should see "ORD-010"
    And I should not see "ORD-011"
    And I should see "Showing 1-10 of 50"

  Scenario: Navigate to next page
    When I click "Next"
    And wait for AJAX to finish
    Then I should see "ORD-011"
    And I should see "ORD-020"
    And I should not see "ORD-001"
    And I should see "Showing 11-20 of 50"

  Scenario: Navigate to specific page
    When I click "3"
    And wait for AJAX to finish
    Then I should see "ORD-021"
    And I should see "Showing 21-30 of 50"

  Scenario: Navigate to previous page
    When I click "2"
    And wait for AJAX to finish
    Then I should see "ORD-011"
    When I click "Previous"
    And wait for AJAX to finish
    Then I should see "ORD-001"
    And I should see "Showing 1-10 of 50"

  Scenario: Change items per page
    When I select "25" from "Items per page"
    And wait for AJAX to finish
    Then I should see "ORD-001"
    And I should see "ORD-025"
    And I should see "Showing 1-25 of 50"

  Scenario: Last page navigation
    When I click "Last"
    And wait for AJAX to finish
    Then I should see "ORD-050"
    And I should not see "Next"
```

**How It Works:**

The scenarios use sequential reference numbers (ORD-001 through ORD-050) as both row content and boundary markers. That gives every page a predictable first and last record, so each scenario asserts three things: the new page's first item is present, the old page's items are gone (`should not see "ORD-001"` after clicking Next), and the "Showing X-Y of Z" counter agrees. The counter assertion is the cheapest of the three and catches a surprising number of off-by-one bugs -- a page that shows eleven rows, or a total count that ignores filters.

The duplicate-page bug from the Problem statement is caught by the negative assertion. If page two re-renders page one's data, `I should see "ORD-011"` fails immediately. Without the negative check, a broken offset that appends rows instead of replacing them would pass.

**At scale:** Pagination tests demand seeded volume -- at least two and a half pages of records, created by your seed script with predictable identifiers. Across many projects, put the record count and the per-page size into the seed configuration rather than hardcoding them into scenarios, and standardize one page size across your portfolio's admin UIs if you can. The same six pagination scenarios then run everywhere, and the only thing each project supplies is its own fixture file.

---

## Recipe 95: Testing Table Search

**Problem:** The product search box works perfectly in every demo -- and then a flaky test report lands on your desk: search scenarios pass locally, fail one run in five on CI. The culprit is the debounce. The app waits 300ms after the last keystroke before firing the request, and on a slow CI runner your assertion sometimes runs in that gap, against the unfiltered table. You need scenarios that test the search behavior itself -- matching, case-insensitivity, empty results, clearing -- without racing the debounce timer.

**Solution:**

```gherkin
Feature: Table Search
  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/products"
    And wait for AJAX to finish

  Scenario: Search finds matching products
    When I fill in "Search products" with "laptop"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"
    And I should see "Laptop Air"
    And I should not see "Wireless Mouse"

  Scenario: Search is case insensitive
    When I fill in "Search products" with "LAPTOP"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"

  Scenario: Search with no results
    When I fill in "Search products" with "xyznonexistent"
    And wait for AJAX to finish
    Then I should see "No products found"

  Scenario: Clear search restores all products
    When I fill in "Search products" with "laptop"
    And wait for AJAX to finish
    Then I should not see "Wireless Mouse"

    When I fill in "Search products" with ""
    And wait for AJAX to finish
    Then I should see "Wireless Mouse"
    And I should see "Laptop Pro"

  Scenario: Search with debounce
    When I fill in "Search products" with "wire"
    # Wait for the debounce delay (typically 300ms)
    And wait 1s
    And wait for AJAX to finish
    Then I should see "Wireless Mouse"
    And I should see "Wireless Keyboard"
```

**How It Works:**

Here is the part most people miss: webship-js already knows about the debounce. Through `addInitScript`, it wraps `setTimeout` on every page and counts pending timers in `window.__webshipPendingTimers`, alongside the fetch/XHR counter in `window.__webshipAjaxCount`. When the app schedules a 300ms debounce after your keystroke, that timer counts as "pending" -- and the settle check that runs after each step (ajax count zero AND pending timers zero AND DOM quiet for at least 250ms) will not release the next step until the debounce fires, the request completes, and the table re-renders. The race condition from the Problem statement is exactly what this machinery exists to kill.

So in most suites the explicit `wait 1s` in the last scenario is redundant -- the auto-settle covers it. It earns its keep only for apps that debounce with something the timer wrapper cannot see (a requestAnimationFrame loop, a third-party widget with its own scheduler). Try removing it; if the scenario stays green for a week of CI runs, leave it out.

The case-insensitivity scenario is worth keeping even though it looks trivial. Search case-sensitivity depends on the database collation, not the application code, so it is one of the few behaviors that genuinely differs between environments -- a SQLite-backed dev setup and a PostgreSQL production can disagree.

---

## Recipe 96: Testing Bulk Actions

**Problem:** Bulk actions are where a small bug becomes a big incident. An admin selects three products to archive, the checkbox state desyncs after a filter change, and "Apply" archives the whole catalog. These are the most destructive buttons in your application, usually behind the least-tested code path -- and the confirmation modal that is supposed to be the safety net often displays the wrong count. If you automate only one thing on an admin table, automate this.

**Solution:**

```gherkin
Feature: Bulk Table Actions
  As an admin
  I want to perform actions on multiple rows at once
  So that I can manage data efficiently

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/products"
    And wait for AJAX to finish

  Scenario: Select all rows and delete
    When I check "Select All"
    And I press "Delete Selected"
    And wait for the modal to appear
    Then I should see "Delete 10 products?" in the modal
    When I click "Confirm" in the modal
    And wait for AJAX to finish
    Then I should see "10 products deleted"

  Scenario: Select individual rows
    When I check "Laptop Pro"
    And I check "Wireless Mouse"
    And I press "Delete Selected"
    And wait for the modal to appear
    Then I should see "Delete 2 products?" in the modal
    When I click "Cancel" in the modal
    And wait for the modal to disappear
    # Products should still be there
    Then I should see "Laptop Pro"
    And I should see "Wireless Mouse"

  Scenario: Bulk status change
    When I check "Select All"
    And I select "Archive" from "Bulk Actions"
    And I press "Apply"
    And wait for AJAX to finish
    Then I should see "10 products archived"

  Scenario: Export selected rows
    When I check "Laptop Pro"
    And I check "Wireless Mouse"
    And I check "USB Cable"
    And I press "Export Selected"
    And wait for AJAX to finish
    Then I should see "3 products exported"
```

**How It Works:**

`When I check "Laptop Pro"` uses the same label-first resolution as form fields: it looks for a checkbox whose associated label (or nearby text) matches "Laptop Pro". Most row-selection checkboxes are wired to the row's title through a label or aria-label, so this works without selectors. If your table's checkboxes are anonymous (`<input type="checkbox">` with no label), ask the developers to add aria-labels -- it fixes your tests and their accessibility audit in one change.

The real assertions in these scenarios are the counts. `"Delete 10 products?"` in the modal and `"10 products deleted"` afterward verify two different things: the first proves the selection state was correct *before* the destructive action, the second proves the backend acted on exactly that selection. The checkbox-desync bug from the Problem statement fails the first assertion -- which is the one that fires before any data is harmed. And note the Cancel scenario: proving that Cancel does nothing is as important as proving Confirm does something.

**At scale:** Tag these scenarios (something like `@destructive`) and make your CI seed a fresh dataset before they run -- a bulk-delete scenario that runs against shared test data will sabotage every scenario after it. On a multi-project CI farm this is a pipeline-ordering question for your DevOps team: destructive suites run last, or against their own database snapshot, and never against an environment another team is mid-run on.

---

## Recipe 97: Testing Expandable/Collapsible Rows

**Problem:** The orders table got a nice UX upgrade: click a row and it expands to show the line items inline, no page navigation. Then a JavaScript refactor breaks the toggle in one direction -- rows expand but never collapse, or the second click navigates to a detail page instead. Nobody notices in review because everyone tests the expand, never the collapse. Toggle interactions need both directions tested, every time.

**Solution:**

```gherkin
Feature: Expandable Table Rows
  As a user viewing orders
  I want to expand rows for details
  So that I can see order items without leaving the list

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/orders"
    And wait for AJAX to finish

  Scenario: Expand a row to see details
    Then I should see "ORD-001"
    And I should not see "Order Items"

    When I click "ORD-001"
    And wait 1s
    Then I should see "Order Items"
    And I should see "Laptop Pro x 1"
    And I should see "USB Cable x 2"

  Scenario: Collapse an expanded row
    When I click "ORD-001"
    And wait 1s
    Then I should see "Order Items"

    When I click "ORD-001"
    And wait 1s
    Then I should not see "Order Items"

  Scenario: Multiple rows can be expanded
    When I click "ORD-001"
    And wait 1s
    When I click "ORD-002"
    And wait 1s
    Then I should see "Laptop Pro x 1"
    And I should see "Wireless Mouse x 3"
```

**How It Works:**

The first scenario sets up the assertion before the action: `I should not see "Order Items"` confirms the detail panel is genuinely hidden before the click, so the post-click `I should see` proves the click caused the change. Without that baseline, an expand panel that renders open by default would pass the test while the toggle is broken.

The `wait 1s` after each click is there for the expand animation. If the row content is inserted into the DOM by JavaScript, webship-js's MutationObserver (tracked through `window.__webshipLastMutation`) already holds the next step until the DOM has been quiet for 250ms -- so for most implementations the auto-settle covers it and the explicit wait can go. Where it still helps is CSS-only expansion: content that is already in the DOM but revealed by a height transition produces no mutations and no AJAX, so nothing tells webship-js to wait. Know which kind your app uses before deleting the wait.

The multi-expand scenario answers a product question, not just a technical one: is this an accordion (opening one row closes the other) or independent panels? Either behavior can be correct -- but the test pins down which one was intended, so a refactor that silently changes it gets caught.

---

## Recipe 98: Testing Empty State Tables

**Problem:** Your demo environment always has data, so nobody on the team has seen the orders page the way a brand-new customer sees it: empty. Then the first real signup lands on a bare table header with zero rows -- or worse, a JavaScript error because the rendering code assumed at least one record. Empty states are the first screen every new user meets, and the last screen anyone on the team looks at. That mismatch is exactly why they break.

**Solution:**

```gherkin
Feature: Empty Table State
  Scenario: New account has no orders
    Given I am a logged in user with the "newuser" user
    When I go to "/orders"
    And wait for AJAX to finish
    Then I should see "No orders yet"
    And I should see "Start shopping to see your orders here"
    And I should see "Browse Products"

  Scenario: Filter produces no results
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And wait for AJAX to finish
    And I fill in "Search" with "nonexistentuser12345"
    And wait for AJAX to finish
    Then I should see "No users found"
    And I should see "Try different search terms"

  Scenario: Empty state has call to action
    Given I am a logged in user with the "admin" user
    When I go to "/admin/categories"
    And wait for AJAX to finish
    Then I should see "No categories yet"
    And I should see "Create your first category"
    When I press "Create Category"
    And wait for AJAX to finish
    Then I should see "New Category"
```

**How It Works:**

```
Table State Flow:
┌────────────────────────────────────────────────────┐
│                                                    │
│  Has data?                                         │
│     │                                              │
│     ├── Yes ──► Show table with rows              │
│     │          ┌────────┬────────┬──────────┐     │
│     │          │ Name   │ Email  │ Actions  │     │
│     │          ├────────┼────────┼──────────┤     │
│     │          │ John   │ j@e.c  │ Edit Del │     │
│     │          └────────┴────────┴──────────┘     │
│     │                                              │
│     └── No ───► Show empty state                  │
│                ┌──────────────────────────────┐   │
│                │                              │   │
│                │   (empty illustration)       │   │
│                │                              │   │
│                │   No orders yet              │   │
│                │   Start shopping to see      │   │
│                │   your orders here.          │   │
│                │                              │   │
│                │   [Browse Products]          │   │
│                │                              │   │
│                └──────────────────────────────┘   │
│                                                    │
└────────────────────────────────────────────────────┘
```

Empty states are important to test because:
- They are the first thing new users see
- A broken empty state creates confusion ("Is the page loading? Is it broken?")
- The call-to-action button guides users to the next step
- They should never show a broken table with zero rows and no explanation

The three scenarios cover the three distinct ways a table ends up empty: a genuinely new account (`newuser` -- which means your seed data must include a user with no records, a fixture most teams forget to create), a filter that matches nothing, and an empty collection with a creation flow. They are different code paths in most apps -- the "no data at all" branch and the "no data matching your filter" branch are usually separate templates -- so one passing scenario does not vouch for the others.

**At scale:** Empty-state rendering is where unhandled JavaScript errors love to hide (`cannot read property 'length' of undefined`). Run these scenarios with JS error capture on -- `WEBSHIP_JS_ERROR_MODE=fail`, or tag the scenarios `@js-fail` -- so a console error on the empty page fails the test even when the visible message happens to render. That single setting turns three modest scenarios into a crash detector for every new account's first session.

---

## Recipe 99: Testing Inline Editing in Tables

**Problem:** Inline editing is a gift to admins and a trap for testers. The price cell turns into an input, the admin types a new value, hits Save -- and sometimes the cell shows the new price while the database still holds the old one, because the optimistic UI updated before the failed request came back. Your test clicks Save, sees "$1,099.99", and goes green on a value that never persisted. Inline edit tests have to distrust what the cell shows immediately after saving.

**Solution:**

```gherkin
Feature: Inline Table Editing
  As an admin
  I want to edit values directly in the table
  So that I can make quick changes without opening a form

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/products"
    And wait for AJAX to finish

  Scenario: Edit price inline
    When I click "Edit" in the "Laptop Pro" row
    And wait 1s
    And I fill in "Price" with "1099.99"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "$1,099.99" in the "Laptop Pro" row

  Scenario: Edit status inline
    When I click "Edit" in the "Laptop Pro" row
    And wait 1s
    And I select "Out of Stock" from "Status"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Out of Stock" in the "Laptop Pro" row

  Scenario: Cancel inline edit
    When I click "Edit" in the "Laptop Pro" row
    And wait 1s
    And I fill in "Price" with "0.01"
    And I press "Cancel"
    And wait 1s
    Then I should see "$999.99" in the "Laptop Pro" row
    And I should not see "$0.01"
```

**How It Works:**

When the Edit click swaps the cell into edit mode, the new `<input>` is inserted by JavaScript -- and `I fill in "Price"` finds it by label-first resolution the same as any form field, falling back to id, name, and placeholder. Inline editors frequently skip visible labels, so check what the input actually carries; a `placeholder="Price"` is enough for the step to resolve it. After Save, the AfterStep settle waits for the PATCH/POST to complete before the row assertion runs.

To close the optimistic-UI gap from the Problem statement, do not stop at asserting the cell text after Save. Add a `When I reload the page` followed by the same row assertion: the reload forces a fresh read from the backend, so the test verifies what persisted rather than what the JavaScript painted. It costs one extra page load and converts a cosmetic check into a data-integrity check.

The Cancel scenario deserves its place: it asserts both that the old value survived ($999.99) and that the typed-but-discarded value left no trace ($0.01). An inline editor that "cancels" the UI but still fires the save request is a real bug class, and only the follow-up assertions catch it.

---

## Recipe 100: Testing Drag-and-Drop Reordering

**Problem:** The menu builder lets editors drag rows to reorder navigation, and the ticket says "automate it." Two weeks later the drag test passes on Chromium, fails on WebKit, and passes-or-fails on Firefox depending on the runner's mood -- because HTML5 drag events, mouse event timing, and the drag library's own thresholds all interact differently per browser. Meanwhile the actual business requirement ("editors can change the menu order and it persists") could have been green on day one through the Move Up/Move Down buttons.

**Solution:**

Drag-and-drop requires a custom step definition since webship-js does not include a built-in drag step:

```gherkin
Feature: Table Row Reordering
  Scenario: Reorder items by drag and drop
    Given I am a logged in user with the "admin" user
    When I go to "/admin/menu-items"
    And wait for AJAX to finish
    Then I should see "Home"
    And I should see "About"
    And I should see "Contact"

    # Use move up/down buttons if available (easier to test)
    When I click "Move Up" in the "Contact" row
    And wait for AJAX to finish
    Then I should see "Order saved"

  Scenario: Reorder with up/down buttons
    Given I am a logged in user with the "admin" user
    When I go to "/admin/menu-items"
    And wait for AJAX to finish
    When I click "Move Down" in the "Home" row
    And wait for AJAX to finish
    Then I should see "Order updated"
```

**Custom Step Definition for Drag-and-Drop:**

```javascript
// tests/step-definitions/custom-drag.js
const { When } = require('@cucumber/cucumber');

When('I drag {string} to {string}', async function (source, target) {
  const sourceEl = this.page.getByText(source).first();
  const targetEl = this.page.getByText(target).first();

  const sourceBox = await sourceEl.boundingBox();
  const targetBox = await targetEl.boundingBox();

  await this.page.mouse.move(
    sourceBox.x + sourceBox.width / 2,
    sourceBox.y + sourceBox.height / 2
  );
  await this.page.mouse.down();
  await this.page.mouse.move(
    targetBox.x + targetBox.width / 2,
    targetBox.y + targetBox.height / 2,
    { steps: 10 }
  );
  await this.page.mouse.up();
});
```

**How It Works:**

The custom step drives raw mouse events through Playwright's mouse API: move to the source row's center, press down, glide to the target in ten intermediate steps, release. The `steps: 10` option is not decoration -- drag libraries typically ignore a teleporting cursor and only engage after seeing continuous movement past a threshold, so the intermediate move events are what make the library believe a human is dragging. Even then, libraries built on HTML5 drag events (`dragstart`/`drop`) rather than mouse events will not respond to this at all; those need `dispatchEvent`-based workarounds that are even more fragile.

This is why the Gherkin scenarios lean on Move Up/Move Down buttons instead. The buttons hit the same backend reorder endpoint and the same persistence logic -- the part that actually matters to the business -- through a click step that is reliable on every browser. The drag gesture itself becomes a thin UI layer you verify manually or in a single quarantined scenario.

**Pro Tip:** Drag-and-drop is notoriously difficult to test reliably across browsers and frameworks. If your application supports both drag-and-drop and button-based reordering (move up/move down), test the button-based approach in your automated tests. It is more reliable and tests the same business logic. Reserve drag-and-drop testing for manual testing or a dedicated drag-drop test with the custom step definition above.

**At scale:** If several of your projects need drag testing, do not let each suite grow its own copy of this custom step. Put it in a shared step-definition package your projects install, and tag every scenario that uses it (say `@drag`) so CI can run those scenarios on Chromium only and skip them on the browsers where they flake. One shared implementation plus one tag convention beats five slightly-different copies producing five different flake patterns in your nightly reports.

---

## Testing Lists (Non-Table Patterns)

Not all data lists use `<table>` elements. Many modern applications use card layouts, list groups, or grid layouts. The same principles apply:

```
Table Layout:                    Card Layout:
┌────────┬────────┬───────┐     ┌──────────┐ ┌──────────┐
│ Name   │ Price  │ Action│     │ Laptop   │ │ Mouse    │
├────────┼────────┼───────┤     │ $999     │ │ $29      │
│ Laptop │ $999   │ Edit  │     │ [Edit]   │ │ [Edit]   │
│ Mouse  │ $29    │ Edit  │     └──────────┘ └──────────┘
└────────┴────────┴───────┘

List Group Layout:               Grid Layout:
┌─────────────────────────┐     ┌───────┬───────┬───────┐
│ ● Laptop Pro    $999 →  │     │ Item 1│ Item 2│ Item 3│
├─────────────────────────┤     ├───────┼───────┼───────┤
│ ● Wireless Mouse $29 →  │     │ Item 4│ Item 5│ Item 6│
├─────────────────────────┤     └───────┴───────┴───────┘
│ ● USB Cable     $9   →  │
└─────────────────────────┘
```

## Recipe 101: Testing Card-Based Lists

**Problem:** The design team replaces the products table with a card grid, and overnight half your table tests stop making sense -- there are no `<tr>` elements left, so every `in the "X" row` step fails with "row not found". The data and the behavior have not changed; only the markup has. This is the third project where it happens, and each time someone rewrites the tests from scratch. There is a cheaper way through.

**Solution:**

```gherkin
Feature: Card List
  Scenario: Product cards display correctly
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"
    And I should see "$999"
    And I should see "Wireless Mouse"
    And I should see "$29"

  Scenario: Click a card to view details
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    And I click "Laptop Pro"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"
    And I should see "Detailed specifications"
    And I should see "Add to Cart"

  Scenario: Card actions
    Given I am a logged in user with the "admin" user
    When I go to "/admin/products"
    And wait for AJAX to finish
    And I press "Edit"
    And wait for AJAX to finish
    Then I should see "Edit Product"
```

**How It Works:**

For card-based layouts, the `in the "X" row` steps will not work because there are no `<tr>` elements -- the row steps query `tr` specifically and filter by text, so a `<div class="card">` is invisible to them. Instead, use the general `should see`, `click`, and `press` steps. If you need to interact with a specific card's action button, two approaches:

1. **Click the card text first** to navigate to its detail page, then interact there
2. **Use custom step definitions** that scope actions to a card container:

```javascript
// tests/step-definitions/custom-card.js
const { When, Then } = require('@cucumber/cucumber');

When('I click {string} on the {string} card', async function (action, cardName) {
  const card = this.page.locator('.card, .product-card, [class*="card"]')
    .filter({ hasText: cardName })
    .first();
  await card.locator('a, button, [role="button"]')
    .filter({ hasText: action })
    .first()
    .click();
});
```

The custom step mirrors the row-step strategy exactly -- find the container by its text, then click inside it -- it just swaps `tr` for the card container's selector. That symmetry is the lesson: scoped interaction is a pattern, not a table feature.

**At scale:** The fragile part of the custom step is the container selector (`.card, .product-card, [class*="card"]`) -- every project's design system names it differently. Keep that selector out of the step code: register it as a named selector in a per-project JSON file under `tests/selectors/` (loaded automatically through `worldParameters.selectors.files`), and have the shared step look the name up. Then twenty projects share one step definition, and onboarding a new project means writing one JSON entry, not forking a step file.

---

## Recipe 102: Testing Data Table with Scenario Outline

**Problem:** Verifying four users means eight assertions; verifying forty means eighty -- and the feature file turns into copy-paste sludge nobody reviews carefully. Then a product owner asks "which users do we actually verify?" and the honest answer is buried across three hundred lines of near-identical scenarios. You want the expected data in one readable table that a non-engineer can review in a pull request, and the repetition generated for you.

**Solution:**

```gherkin
Feature: Data Verification
  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin/users"
    And wait for AJAX to finish

  Scenario Outline: User data is correct in table
    Then I should see "<role>" in the "<email>" row
    And I should see "<status>" in the "<email>" row

    Examples:
      | email              | role   | status  |
      | john@example.com   | Admin  | Active  |
      | jane@example.com   | Editor | Active  |
      | bob@example.com    | Viewer | Blocked |
      | alice@example.com  | Admin  | Active  |

  Scenario Outline: Action buttons are available for each user
    When I click "View" in the "<email>" row
    And wait for AJAX to finish
    Then I should see "<name>"
    When I move backward one page

    Examples:
      | email              | name        |
      | john@example.com   | John Smith  |
      | jane@example.com   | Jane Doe    |
      | bob@example.com    | Bob Wilson  |
```

**How It Works:**

Scenario Outline generates one test scenario per row in the Examples table. This is powerful for table verification because you can test every row without writing repetitive scenarios.

```
Scenario Outline generates:

Row 1: Then I should see "Admin" in the "john@example.com" row
        And I should see "Active" in the "john@example.com" row

Row 2: Then I should see "Editor" in the "jane@example.com" row
        And I should see "Active" in the "jane@example.com" row

Row 3: Then I should see "Viewer" in the "bob@example.com" row
        And I should see "Blocked" in the "bob@example.com" row

Row 4: Then I should see "Admin" in the "alice@example.com" row
        And I should see "Active" in the "alice@example.com" row
```

Each generated scenario is independent: it gets its own Background run (login, navigate, settle) and reports its own pass/fail line. That granularity is the win over one giant scenario with eight assertions -- when bob@example.com's role regresses, the report says exactly which example failed, instead of failing one mega-scenario and hiding the other seven results behind it. The price is runtime: four examples means four logins and four page loads. Keep outlines for data variation; do not use them to re-run identical behavior.

There is a quieter benefit for the people who do not write tests. The Examples table *is* the acceptance criteria -- a product owner can read it, spot that alice should be an Editor not an Admin, and comment on the pull request without parsing a line of automation code. When your BA writes the Examples table and QA writes the steps above it, sign-off stops being an act of trust.

**At scale:** The Examples table is data, and data wants to live with the seed fixtures. If the same user matrix is seeded in every project, generate or copy the Examples block from the same source the seed script uses -- one YAML or JSON file feeding both. The moment seed data and Examples tables are maintained by hand in two places, they will drift, and you will spend an afternoon debugging a "failure" that is just a stale expectation.

---

## Table Testing Quick Reference

```
┌───────────────────────────────────────────────────────────────────┐
│ What You Want                    │ Step to Use                    │
├───────────────────────────────────────────────────────────────────┤
│ Verify text exists on page       │ Then I should see "text"       │
│ Verify text in specific row      │ Then I should see "text" in    │
│                                  │   the "identifier" row         │
│ Verify text NOT in a row         │ Then I should not see "text"   │
│                                  │   in the "identifier" row      │
│ Click action in a row            │ When I click "Edit" in the     │
│                                  │   "identifier" row             │
│ Sort by column                   │ When I click "Column Name"     │
│ Filter by text                   │ When I fill in "Search"        │
│                                  │   with "query"                 │
│ Filter by dropdown               │ When I select "value" from     │
│                                  │   "filter"                     │
│ Navigate to next page            │ When I click "Next"            │
│ Select a row (checkbox)          │ When I check "identifier"      │
│ Select all rows                  │ When I check "Select All"      │
│ Empty state                      │ Then I should see "No data"    │
│ Wait after table action          │ And wait for AJAX to finish    │
└───────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- Tables are the core of data-driven applications
- Use `in the "X" row` steps to scope assertions and actions to specific rows
- Choose unique row identifiers: emails, IDs, reference numbers
- Always verify both presence AND absence when testing filters
- Test the complete CRUD lifecycle through the table
- Sort testing verifies the first/last visible item after sorting
- Pagination testing navigates forward, backward, and to specific pages
- Bulk actions: select rows → choose action → confirm → verify result
- Empty states should show helpful messages with calls to action
- Card layouts need different strategies than table layouts
- Use Scenario Outline + Examples for efficient multi-row verification
- Always `wait for AJAX to finish` after actions that trigger server requests

Next: Authentication and Authorization Testing recipes.

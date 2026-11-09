# Appendix B: webship-js Step Definition Quick Reference

---

This is the cheat sheet. Print it. Pin it to your wall. Open it in a second monitor. These are all the built-in steps that webship-js gives you out of the box, organized by what you are trying to do.

---

## Navigation

| Step | What It Does |
|------|-------------|
| `Given I am on homepage` | Navigate to the base URL |
| `Given I am on "{path}"` | Navigate to a specific path |
| `When I go to "{path}"` | Navigate to a path (relative or absolute) |
| `When I go to the homepage` | Navigate to the base URL |
| `When I move backward one page` | Browser back button |
| `When I move forward one page` | Browser forward button |
| `When I reload the page` | Refresh the current page |

---

## Waiting

| Step | What It Does |
|------|-------------|
| `And wait for the page to load` | Wait for `document.readyState` and `networkidle` |
| `And wait for AJAX to finish` | Wait for all XHR/Fetch requests to complete |
| `And wait {int}s` | Wait a specific number of seconds |
| `And wait for the modal to appear` | Wait for a visible modal/dialog |
| `And wait for the modal to close` | Wait for all modals to be hidden |

---

## Clicking

| Step | What It Does |
|------|-------------|
| `When I click "{text}"` | Click a link or element by visible text |
| `When I press "{text}"` | Click a button by visible text or value |
| `When I click "{text}" in the "{row}" row` | Click a link in a specific table row |
| `When I press "{text}" in the "{row}" row` | Click a button in a specific table row |
| `When I click "{text}" in the modal` | Click a link inside the currently visible modal |
| `When I click "{text}" in the modal content` | Click inside the modal body |
| `When I press "{text}" in the modal` | Click a button inside the modal |

---

## Forms

| Step | What It Does |
|------|-------------|
| `When I fill in "{field}" with "{value}"` | Type text into an input field |
| `When I select "{value}" from "{field}"` | Select an option from a dropdown |
| `When I additionally select "{value}" from "{field}"` | Select additional option (multi-select) |
| `When I check "{field}"` | Check a checkbox |
| `When I uncheck "{field}"` | Uncheck a checkbox |
| `When I attach the file "{path}" to "{field}"` | Upload a file to a file input |

---

## Assertions -- Visibility

| Step | What It Does |
|------|-------------|
| `Then I should see "{text}"` | Assert text is visible on the page |
| `Then I should not see "{text}"` | Assert text is NOT visible on the page |
| `Then I should see "{text}" in the "{region}" region` | Assert text in a specific region |
| `Then I should not see "{text}" in the "{region}" region` | Assert text NOT in a region |
| `Then I should see "{text}" in the modal` | Assert text inside the visible modal |

---

## Assertions -- Elements

| Step | What It Does |
|------|-------------|
| `Then the "{field}" field should contain "{value}"` | Assert input field has a value |
| `Then the "{field}" field should not contain "{value}"` | Assert input field does NOT have a value |
| `Then the checkbox "{field}" should be checked` | Assert checkbox is checked |
| `Then the checkbox "{field}" should not be checked` | Assert checkbox is unchecked |
| `Then the "{field}" select should have "{option}" selected` | Assert selected dropdown value |

---

## Assertions -- URL and Title

| Step | What It Does |
|------|-------------|
| `Then I should be on "{path}"` | Assert current URL matches |
| `Then the page title should contain "{text}"` | Assert page title contains text |

---

## Scrolling

| Step | What It Does |
|------|-------------|
| `When I scroll to top` | Scroll to the top of the page |
| `When I scroll to bottom` | Scroll to the bottom of the page |
| `When I scroll to "{selector}"` | Scroll to a specific element |

---

## Modal and Dialog

| Step | What It Does |
|------|-------------|
| `Then I should see a modal` | Assert a modal/dialog is visible |
| `Then I should not see a modal` | Assert no modal is visible |
| `When I close the modal` | Click the close button in the modal |
| `When I fill in "{field}" with "{value}" in the modal` | Fill a field inside the modal |

---

## Screenshots

| Step | What It Does |
|------|-------------|
| `Then I take a screenshot` | Capture full-page screenshot |
| `Then I take a screenshot of "{selector}"` | Capture screenshot of specific element |

---

## API Testing (webship-api.js)

| Step | What It Does |
|------|-------------|
| `Given the API base URL is "{url}"` | Set the base URL for API requests |
| `Given I am authenticating as "{user}" with "{pass}" password` | Set Basic Auth |
| `When I send a GET request to "{path}"` | Send GET request |
| `When I send a POST request to "{path}" with body:` | Send POST with JSON body |
| `When I send a PUT request to "{path}" with body:` | Send PUT with JSON body |
| `When I send a PATCH request to "{path}" with body:` | Send PATCH with JSON body |
| `When I send a DELETE request to "{path}"` | Send DELETE request |
| `Then the API response code should be {int}` | Assert HTTP status code |
| `Then the response should be valid JSON` | Assert response is parseable JSON |
| `Then the API response should contain "{text}"` | Assert response body contains text |
| `Then the JSON response should have "{key}" equal to "{value}"` | Assert JSON property value |
| `Then the JSON response should have property "{key}"` | Assert JSON property exists |
| `Then the JSON response should not have property "{key}"` | Assert JSON property absent |

---

## Environment Variables

| Variable | Default | What It Does |
|----------|---------|-------------|
| `BROWSER` | `chromium` | Browser engine: `chromium`, `firefox`, `webkit` |
| `HEADLESS` | `true` | Run headed: `HEADLESS=false` |
| `SLOW_MO` | `300` | Milliseconds delay between actions |
| `LAUNCH_URL` | from config | Override the base URL |

---

## Common Patterns

### Login and Do Something

```gherkin
Given I am a logged in user with the "admin" user
When I go to "/admin/products"
And wait for AJAX to finish
Then I should see "Products"
```

### Fill a Form and Submit

```gherkin
When I fill in "Name" with "Jane Doe"
And I fill in "Email" with "jane@example.com"
And I select "Premium" from "Plan"
And I check "I agree to the terms"
And I press "Submit"
And wait for AJAX to finish
Then I should see "Registration complete"
```

### Test a Table

```gherkin
Then I should see "Laptop Pro"
When I click "Edit" in the "Laptop Pro" row
And wait for AJAX to finish
And I fill in "Price" with "899.99"
And I press "Save"
And wait for AJAX to finish
Then I should see "Product updated"
```

### Test a Modal

```gherkin
When I press "Delete"
And wait for the modal to appear
Then I should see "Are you sure?" in the modal
When I press "Confirm" in the modal
And wait for AJAX to finish
Then I should see "Deleted"
```

### API Call

```gherkin
Given the API base URL is "http://localhost:8080"
When I send a GET request to "/api/products"
Then the API response code should be 200
And the JSON response should have property "data"
```

---

## Web-First Assertions (with `within N seconds`)

| Step | What It Does |
|------|-------------|
| `Then "sel" should be visible within {N} seconds` | Poll until visible |
| `Then "sel" should be hidden within {N} seconds` | Poll until hidden |
| `Then "sel" should be attached` | Element present in DOM |
| `Then "sel" should be focused` | Element has keyboard focus |
| `Then "sel" should be enabled` / `disabled` | Interactive state |
| `Then "sel" should be editable` | Editable (not readonly/disabled) |
| `Then "sel" should contain text "X"` | Substring match |
| `Then "sel" should have text "X"` | Exact text match |
| `Then "sel" should have value "X"` | Input value |
| `Then "sel" should have class "X"` | Class present |
| `Then "sel" should have attribute "X" with value "Y"` | Attribute match |
| `Then "sel" should have a count of {N}` | Match count |
| `Then "sel" should be in the viewport` | Element in visible viewport |
| `Then the "X" button should be visible` | Role-aware |

Default budget 5s. Override with trailing `within N seconds`.

---

## Iframes

| Step | What It Does |
|------|-------------|
| `When I switch to the iframe "name"` | Enter iframe by `name` attr |
| `When I switch to the iframe with title "X"` | Enter iframe by `title` |
| `When I switch to iframe with locator "sel"` | Enter iframe by CSS |
| `When I switch to the root document` | Return to top-level page |
| `When I click "X" inside the iframe` | Click inside current iframe |
| `When I fill in "X" with "Y" inside the iframe` | Fill inside iframe |
| `Then I should see "X" inside the iframe` | Text assertion inside iframe |

---

## Clock and Time Mocking

| Step | What It Does |
|------|-------------|
| `Given the system time is "X"` | Freeze in-page clock at instant |
| `When I advance the clock by {N} ms` | Step clock forward (also: seconds, minutes) |
| `When I pause the clock` | Stop clock progression |
| `When I resume the clock` | Resume real-time |
| `When I wait until pending timers settle` | Fast-forward all queued timers |

---

## Network Mocking

| Step | What It Does |
|------|-------------|
| `Given the URL "X" is blocked` | Block request to URL |
| `Given the URL "X" is delayed by {N} ms` | Delay response |
| `Given the URL "X" returns status {N}` | Mock status code |
| `Given the URL "X" returns status {N} with body "Y"` | Mock status + body |
| `Given the URL "X" returns the JSON:` | Mock JSON response |
| `Given the network is offline` / `online` | Toggle browser online state |

## Network Recording

| Step | What It Does |
|------|-------------|
| `Given I start recording network requests` | Start request recorder |
| `Then a {METHOD} request to "X" should have been made` | Assert request fired |
| `Then a request to "X" should have been made` | Method-less assert |
| `Then no request to "X" should have been made` | Negation |

---

## File Downloads and Video

| Step | What It Does |
|------|-------------|
| `When I download the file from the link "X"` | Trigger download from link |
| `When I download the file from the URL "X"` | Trigger download from URL |
| `Then the downloaded file name should be "X"` | Exact filename |
| `Then the downloaded file name should contain "X"` | Substring filename |
| `Then the downloaded file should contain:` | Substring content |
| `Then the downloaded file should be a zip archive containing the following files named:` | ZIP entry check |
| `Then the downloaded file should be a zip archive not containing the following files partially named:` | Negative ZIP check |
| `Given I start video recording` | Begin video capture |
| `When I stop video recording` | Stop video capture |
| `When I save the current video as "X"` | Save with custom name |

---

## XML Response

| Step | What It Does |
|------|-------------|
| `Then the response should be in XML format` | Detect XML response |
| `Then the XML element "xpath" should exist` | Element exists |
| `Then the XML element "xpath" should be equal to "X"` | Element value |
| `Then the XML element "xpath" should contain "X"` | Element substring |
| `Then the XML element "xpath" should have {N} element(s)` | Element count |
| `Then the XML attribute "X" on element "xpath" should be equal to "Y"` | Attribute value |
| `Then the XML should use the namespace "X"` | Namespace check |

---

## YAML Response

| Step | What It Does |
|------|-------------|
| `Then the response should be in YAML format` | Detect YAML response |
| `Then the YAML element "path" should exist` | Element exists |
| `Then the YAML element "path" should be equal to "X"` | Value match |
| `Then the YAML value at "path" should be of type "X"` | Type check |
| `Then the YAML value at "path" should be greater than {N}` | Numeric range |
| `Then the YAML array at "path" should contain an item where "K" is "V"` | Array predicate |
| `Then the YAML keys at "path" should be exactly "K1,K2,K3"` | Strict keys |
| `Then the YAML response should have {N} document(s)` | Multi-document count |
| `Then the YAML should match JSON Schema "path"` | Schema validate |
| `Then the YAML should have no duplicate keys` | Lint check |

---

## Pointer Actions and Keyboard

| Step | What It Does |
|------|-------------|
| `When I double-click on "X"` | Native dblclick |
| `When I right-click on "X"` | Context menu |
| `When I middle-click on "X"` | Middle button |
| `When I hover over "X"` | Mouse hover |
| `When I tap on "X"` | Mobile touch tap |
| `When I drag "A" to "B"` | Drag-and-drop |
| `When I click on "X" while holding "Shift"` | Click with modifier |
| `When I press the key "X"` | Single key press |
| `When I press the keys "Control+S"` | Key chord |
| `When I press the keys "X" on the element "sel"` | Scoped chord |

---

## Advanced Field Types

| Step | What It Does |
|------|-------------|
| `When I fill in the color field "X" with the value "Y"` | Color picker |
| `When I fill in the datetime field "X" with date "D" and time "T"` | Datetime |
| `When I fill in the date part of the datetime field "X" with "D"` | Date only |
| `When I fill in the start datetime field "X" with date "D" and time "T"` | Range start |
| `When I fill in the end datetime field "X" with date "D" and time "T"` | Range end |
| `When I fill in the multi-value field "X" with the following values:` | Tag chips |
| `When I fill in the WYSIWYG field "X" with the "Y"` | Rich editor |
| `When I select "X" text in "Y" field` | Substring select |
| `When I select from {N} to {M} text in "Y" field` | Range select |
| `When I select all text in "Y" field` | Select all |
| `Then the field "X" should be required` / `empty` / `not empty` | Field state |
| `Then the field "X" should have "valid" state` | Validity state |

---

## JavaScript Console Health

| Step | What It Does |
|------|-------------|
| `Then there should be no JavaScript errors` | Fail on any console error |
| `Then there should be no JavaScript warnings` | Fail on any console warning |
| `Then JavaScript errors should not match "pattern"` | Fail on matching error |
| `Then print JavaScript errors` | Dump errors to stdout |

Modes: `fail` (default), `warn`, `off` -- set via `this.javascriptErrorMode` in a Before hook.

---

## Cookies (Four-Way Matrix)

| Name | Value | Step (existence) |
|------|-------|------------------|
| exact | exact | `a cookie with the name "X" and the value "Y" should exist` |
| exact | contains | `a cookie with the name "X" and a value containing "Y" should exist` |
| contains | exact | `a cookie with a name containing "X" and the value "Y" should exist` |
| contains | contains | `a cookie with a name containing "X" and a value containing "Y" should exist` |
| (any) | (any) | `a cookie with the name "X" should exist` (name-only) |

All have `should not exist` negations. Setters: `the cookie "X" is set to "Y"`, `the cookie "X" is removed`, `all cookies are cleared`.

---

## Storage

| Step | What It Does |
|------|-------------|
| `Given the local storage "X" is set to "Y"` | Seed localStorage key |
| `Given the local storage "X" is removed` | Remove single key |
| `Given local storage is cleared` | Clear all localStorage |
| `Given the session storage "X" is set to "Y"` | Seed sessionStorage key |
| `Given the session storage "X" is removed` | Remove single key |
| `Given session storage is cleared` | Clear all sessionStorage |

---

## Named Selectors

| Step | What It Does |
|------|-------------|
| `Given I add "name" selector for "css" css selector` | Inline CSS selector |
| `Given I add "name" selector for "xpath" xpath selector` | Inline XPath selector |
| `Given I define css selectors:` (with table) | Bulk CSS |
| `Given I define xpath selectors:` (with table) | Bulk XPath |
| `Given I add selectors from "file.json" file` | Load from file |
| `Then print css selectors` / `print xpath selectors` | Dump registry |

---

## Spatial & Layout

| Step | What It Does |
|------|-------------|
| `Then the element "X" should be displayed within a viewport` | In current viewport |
| `Then the element "X" should be at the top of the viewport` | Sticky/pinned |
| `Then the element "X" should be centered in the viewport` | Centred element |
| `Then the element "X" should appear after the element "Y"` | DOM ordering |
| `Then the text "X" should appear after the text "Y"` | Text ordering |
| `Then "X" should contain cssproperty "prop" with value "val"` | Computed style |

---

## CI/CD Providers Shipped

GitHub Actions · GitLab CI · Bitbucket Pipelines · CircleCI · Travis CI · Jenkins · Azure Pipelines · AWS CodeBuild · Google Cloud Build · TeamCity (Kotlin DSL) · Semaphore · Drone CI · Woodpecker CI · Forgejo Actions · Harness CI · Bamboo Data Center · Codefresh · Octopus Deploy (CD-only) · Codeship (retired)

See Chapter 45 for full per-provider configs.

---

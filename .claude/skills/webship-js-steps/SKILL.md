---
name: webship-js-steps
description: Quick reference for every webship-js step definition installed in the current project — Given / When / Then UI, web-first auto-wait, API + REST, a11y (axe-core), iframe, clock, network mocking, cookies + storage, video, XML / YAML, screenshots, tables, file downloads, JS-error capture, debug. Use when the user asks "what steps exist for X", "list webship-js steps", "step reference", "how do I assert <Y>", "is there a step for <Z>". Filter by category.
argument-hint: '[category]'
arguments: category
allowed-tools: Bash(ls *) Bash(cat *) Bash(grep *) Read Glob Grep
---

# /webship-js-steps — Step definition reference

Display available [webship-js](https://webship.co/docs/webship-js/2.0.x)
step definitions.

`$category` — optional filter. Maps to files in
`node_modules/webship-js/tests/step-definitions/`:

`a11y`, `action`, `api`, `assertion`, `auth`, `clock`, `cookie`, `debug`,
`dialog`, `element`, `field`, `file-download`, `form`, `iframe`, `input`,
`javascript`, `keyboard`, `link`, `metatag`, `modal`, `navigation`,
`network`, `path`, `response`, `responsive`, `rest`, `screenshot`,
`scroll`, `selectors`, `storage`, `table`, `video`, `wait`, `web-first`,
`xml`, `yaml`, or `all` (default).

## Context loading (source of truth)

1. **Installed step files** — list and read what's actually present:

   ```bash
   ls node_modules/webship-js/tests/step-definitions/*.steps.js
   ```

   Each file owns one category. Read the JSDoc above each `Given` / `When` /
   `Then` for the canonical phrasing + at least 5 examples.

2. **Docs at `node_modules/webship-js/docs/`** — topical reference for
   each step family:

   - `04-step-reference.md` — full topical reference.
   - `02-bbr-smart-waits.md` — wait phrasings.
   - `03-selector-registry.md` — selector registry steps + CMS / framework
     presets + canonical key set.
   - `05-web-first-assertions.md` — auto-wait matchers + role-based
     interactions.
   - `06-network-and-dialogs.md` — request stubbing, recording, browser
     dialogs.
   - `07-auth-state.md` — save / restore / clear auth state.
   - `08-clock-mocking.md` — system time + advance.
   - `09-api-testing.md` — API + REST.
   - `10-accessibility.md` — a11y audits + structural checks.
   - `15-tag-conventions.md` — standard tags + CI lanes.
   - `advanced-selectors.md` + `advanced-screenshots.md` — deep dives.

3. **Detailed step pages** at
   `node_modules/webship-js/docs/step-definitions/` (per-step Markdown).

If webship-js isn't installed locally, fetch the same files from
https://github.com/webship/webship-js/tree/2.0.x.

4. **Current project** — include custom steps in the output:

   - `tests/step-definitions/*.js`
   - `tests/features/*.feature` (existing patterns to stay consistent
     with).

Steps below reflect the catalog at the time of writing. **Verify against
the installed source — skip anything that's not there.** Every step
supports `I` / `we` pronouns (optional). `the`, `a`, `an` tokens are
often optional too — check each regex.

## Quick reference

### Context (Given) — `navigation.steps.js` / `selectors.steps.js`

```gherkin
Given I am an anonymous user
Given I am on the homepage
Given I am on "/login"
Given I define css selectors:
  | name         | css selector    |
  | login button | button.login    |
Given I define xpath selectors:
  | name       | xpath                  |
  | page title | //h1[@class='title']   |
```

### Viewport — `selectors.steps.js` + `responsive.steps.js`

```gherkin
Given I am viewing the site on a "xl" screen
Given I am viewing the site on a "xs" device
When I set the viewport to the "md" breakpoint
When I set the viewport width to 1024
When I set the viewport to 1280 by 800
# breakpoints: xs, sm, md, lg, xl (default), xxl, xxxl
```

### Navigation — `navigation.steps.js` / `path.steps.js`

```gherkin
When I go to the homepage
When I go to "/about"
When I reload the page
When I move forward one page
When I move backward one page
When I go back
When I follow "Read more"
Then I should be on the homepage
Then I should be on "/dashboard"
Then the url should match "^/users/\d+$"
Then the path should be "/dashboard"
Then current url should have the "tab" parameter with the "billing" value
```

### Click / press / pointer — `action.steps.js`

```gherkin
When I press "Submit"
When I press "login-btn" by its "id" attribute
When I click "Read more"
When I click "Edit" in the "Order #123" row
When I click on the element "main nav"
When I click the "Sign in" button       # role-based
When I click the "Profile" link
When I click the "Tab 2" tab
When I attach the file "resume.pdf" to "Upload CV"
When I hover over "main nav"
When I double-click on "row 3"
When I right-click on "row 3"
When I drag "card-1" to "drop-zone"
When I tap on "menu"
When I click on "row 3" while holding "Shift"
# positional (requires named selector):
When I click login button
When I click login button, submit button
```

### Form input — `form.steps.js` / `input.steps.js` / `field.steps.js`

```gherkin
When I fill in "email" with "user@example.com"
When I fill in "email-field" with "user@example.com" by its "id" attribute
When I fill in "message" with:
  """
  multi-line value
  """
When I fill in "user@example.com" for "email"
When I fill in the following:
  | Email | user@example.com |
  | Name  | Rajab            |
When I select "Option 1" from "Country"
When I additionally select "Option 2" from "Countries"
When I check "Accept terms"
When I uncheck "Subscribe"
When I select radio button "Male"
# selector-based:
When I fill in the field "#email" with "user@example.com"
When I check the checkbox "#accept"
When I choose the radio button "input[value='yes']"
When I unselect "Option 1" from "#country"
Given browser validation for the form "#contact" is disabled
# specialized:
When I fill in the multi-value field "Tags" with the following values:
  | red |
  | green |
When I fill in the color field "Theme" with the value "#ff0000"
When I fill in the WYSIWYG field "Body" with the "<p>hello</p>"
When I fill in the datetime field "Start" with date "2026-05-14" and time "09:30"
Then the field "#email" should be empty
Then the field "#email" should be required
Then the option "Egypt" should be selected within the select element "#country"
```

### Keyboard — `keyboard.steps.js`

```gherkin
When I press the key "Enter"
When I press the key "Tab" on the element "#email"
When I press the keys "Control+S"
When I press the keys "Control+Shift+P" on the element "body"
```

Keys must be Playwright identifiers — `Enter`, `Tab`, `Escape`, arrows
(`ArrowDown`), function keys, modifier combos (`Control+S`). Bad names
throw `Unknown key: <X>`.

### Scroll — `scroll.steps.js`

```gherkin
When I scroll down
When I scroll down 500
When I scroll to the top
When I scroll to the bottom
When I scroll to top of "main nav"
When I scroll to the element "#section-3"
```

### Waits — `wait.steps.js` (see `docs/02-bbr-smart-waits.md`)

```gherkin
# bounded smart waits (return early on idle)
When I wait 5 seconds
When I wait max of 5 seconds
When I wait for 3 seconds for AJAX to finish

# edge waits
When I wait until the page is loaded
When I wait for AJAX to finish
When I wait until the network is idle
When I wait until the page is interactive
When I wait until pending timers settle

# targeted edge waits
When I wait for "#dashboard" to appear
When I wait for "#loading" to disappear
When I wait for the text "Welcome" to appear
When I wait until the URL contains "/dashboard"
When I wait until the page title contains "Dashboard"
When I wait until 5 elements match ".product-card"
When I wait until at least 3 elements match ".item"

# modal-specific
When I wait for the modal to appear
When I wait for the modal to disappear

# polling text assertion
Then eventually I should see "Done"
Then eventually I should see "Done" within 10 seconds
```

Auto-settle runs after every state-changing step (1500 ms budget).
Disable with `WEBSHIP_AUTO_SETTLE=off`.

### Clock mocking — `clock.steps.js`

```gherkin
Given the system time is "2026-01-01T00:00:00Z"
When I advance the clock by 500 ms
When I advance the clock by 30 seconds
When I advance the clock by 5 minutes
When I pause the clock
When I resume the clock
When I set the system time to "2026-06-15T12:00:00Z"
```

### Network mocking — `network.steps.js` (`docs/06-network-and-dialogs.md`)

```gherkin
Given the URL "**/api/users" returns the JSON:
  """
  [{ "id": 1, "name": "Rajab" }]
  """
Given the URL "/api/users/42" returns status 404
Given the URL "/api/users/42" returns status 500 with body "boom"
Given the URL "/api/slow" is delayed by 2000 ms
Given the URL "/api/secret" is blocked
Given the network is offline
Given the network is online
Given I start recording network requests
Then a request to "/api/users" should have been made
Then a POST request to "/api/users" should have been made
Then no request to "**/google-analytics.com/**" should have been made
```

Patterns use Playwright `page.route()` matching: `**/path`, `*.gif`.

### Storage + cookies — `storage.steps.js` / `cookie.steps.js`

```gherkin
Given the local storage "token" is set to "abc123"
Given local storage is cleared
Given the session storage "tab" is set to "billing"
Given the cookie "session_id" is set to "abc123"
Given all cookies are cleared
Then a cookie with the name "session_id" should exist
Then a cookie with the name "session_id" and a value containing "abc" should exist
Then a cookie with a name containing "session" should exist
```

### Auth state — `auth.steps.js` (`docs/07-auth-state.md`)

```gherkin
Given the basic authentication with the username "admin" and the password "secret"
When I save the auth state to "tests/auth/admin.json"
Given I restore the auth state from "tests/auth/admin.json"
Given I clear the auth state
```

Pattern: one `@auth-setup` scenario per role produces JSON. Every other
scenario restores. No re-login overhead.

### Modals + browser dialogs — `modal.steps.js` / `dialog.steps.js`

```gherkin
Then I should see a modal
Then I should see a modal with title "Confirm delete"
Then I should see "Are you sure?" in the modal
When I click "Yes" in the modal
When I close the modal
# browser alert / confirm / prompt:
Given I will accept the next dialog
Given I will accept the next dialog with "my answer"
Given I will dismiss the next dialog
Given I accept all confirmation dialogs
Then the last dialog message should contain "delete"
Then the last dialog type should be "confirm"
```

### iframe — `iframe.steps.js` (always switch back to root)

```gherkin
When I switch to the iframe "#payment-frame"
When I switch to iframe with locator ".stripe-frame"
When I switch to the iframe with title "Payment form"
When I click "Pay" inside the iframe
When I fill in "Card number" with "4242 4242 4242 4242" inside the iframe
Then I should see "Approved" inside the iframe
When I switch to the root document
```

### Selector registry — `selectors.steps.js`

```gherkin
When I add "header" selector for "header.page-header" css selector
When I add "page title" selector for "//h1[@class='title']" xpath selector
When I add selectors from "homepage-selectors.json" file
Then I print css selectors
```

See `docs/03-selector-registry.md` for the canonical names (`primary
button`, `notice success`, `data table`, etc.) and the 24+ CMS /
framework presets that ship under
`node_modules/webship-js/tests/selectors/`.

### Text assertions — `assertion.steps.js`

```gherkin
Then I should see "Welcome back"
Then I should not see "Error"
Then I should see "Submitted" in the "success message" element
Then I should see text matching "Order #\d+"
Then I should see "Yes" in the "Order #123" row
```

### Element assertions — `element.steps.js`

```gherkin
Then I should see a "submit button" element
Then I should see 3 ".card" elements
Then the "main nav" element should contain "Home"
Then the element ".cta" with the attribute "data-test" and the value "primary" should exist
Then the element "#hero" should be at the top of the viewport
Then the element "#hero" should be displayed
When I trigger the JS event "change" on the element "#country"
When I focus on the element "#email"
```

### Web-first assertions (auto-wait) — `web-first.steps.js` (`docs/05-web-first-assertions.md`)

```gherkin
Then ".submit-btn" should be visible
Then ".submit-btn" should be visible within 5 seconds
Then ".spinner" should not be visible
Then "#submit" should be enabled
Then ".card" should have a count of 3 within 5 seconds
Then "h1" should have text "Welcome"
Then "h1" should contain text "Welcome"
Then "input[name=email]" should have value "user@example.com"
Then "img.logo" should have attribute "alt" with value "Company"
Then "button" should have class "primary"
Then the "Sign in" button should be visible
Then the "Profile" link should be visible within 2 seconds
```

5-second default budget. Override with `within N seconds`. Role-based
interactions support `button`, `link`, `tab`, `menuitem`, `checkbox`,
`radio`, `option`.

### Links — `link.steps.js`

```gherkin
Then the "Read more" link should contain "/articles/42"
Then the "read-more" link should contain "/articles/42" by its "id" attribute
Then the link "Read more" with the href "/articles/42" within the element ".card" should exist
Then the link "Documentation" should be an absolute link
When I click on the link with the title "Open menu"
```

### HTTP response — `response.steps.js`

```gherkin
Then the response status code should be 200
Then the response should contain "OK"
Then the response should contain the header "Content-Type"
Then the response header "Content-Type" should contain the value "application/json"
```

### REST + API — `rest.steps.js` / `api.steps.js` (`docs/09-api-testing.md`)

```gherkin
Given a REST header "Authorization" with value "Bearer xyz"
When I send a REST "POST" request to "https://api.example.com/users" with body:
  """
  { "name": "Rajab" }
  """
Then the REST response status code should be 200

Given the API base URL is "https://api.example.com"
Given I set the following headers:
  | X-Api-Key | abc123           |
  | Accept    | application/json |
When I send a GET request to "/users/:userId"
Then the API response code should be 200
Then the JSON response should have "data.id" equal to 42
Then the JSON response should have property "data.email"
Then the response should be valid JSON
```

### XML / YAML — `xml.steps.js` / `yaml.steps.js`

```gherkin
Given the response content from the file "fixtures/users.xml"
Then the XML element "/users/user[@id='1']" should be equal to "Rajab"
Then the XML attribute "id" on element "/users/user" should be equal to "1"

Given the YAML response content from the file "fixtures/cfg.yml"
Then the YAML element "users.0.name" should be equal to "Rajab"
Then the YAML value at "users.0.age" should be greater than 18
Then the YAML array at "users" should contain an item where "name" is "Rajab"
Then the YAML should match JSON Schema "schemas/user.json"
```

### Tables — `table.steps.js`

```gherkin
Then the table ".orders" should have 5 rows
Then the table ".orders" should contain the following columns:
  | Order # | Customer | Total | Status |
Then the table ".orders" should be sorted by "Total" in "descending" order
Then the "Order #123" row should contain the following:
  | Customer | Rajab |
```

### Accessibility — `a11y.steps.js` (`docs/10-accessibility.md`)

```gherkin
Then the page should pass an accessibility audit
Then the page should pass an accessibility audit at level "AA"
Then the page should pass the accessibility rules "color-contrast,label"
Then the page should pass an accessibility audit excluding ".third-party"
Then the page should have no critical accessibility violations
Then the element "#contact-form" should pass an accessibility audit
# structural (no axe needed)
Then every image should have an alt attribute
Then every form field should have an accessible label
Then the page should have a title
Then the page should declare a language
Then the page should have exactly one h1
Then user zoom should be allowed
```

### Meta tags — `metatag.steps.js`

```gherkin
Then the meta tag should exist with the following attributes:
  | name    | description     |
  | content | Webship-js docs |
```

### Screenshots — `screenshot.steps.js` (`docs/advanced-screenshots.md`)

```gherkin
When I save screenshot
When I save fullscreen screenshot
When I save 1280 x 800 screenshot
When I save screenshot with name "login-filled"
```

### Video — `video.steps.js`

```gherkin
When I start video recording
When I stop video recording
When I save the current video as "checkout-flow"
Then print video path
```

Env: `WEBSHIP_VIDEO` (`off|on|on-failure|tag`), `WEBSHIP_VIDEO_DIR`.

### File downloads — `file-download.steps.js`

```gherkin
When I download the file from the URL "/exports/users.csv"
When I download the file from the link "Export users"
Then the downloaded file should contain:
  """
  id,name
  """
Then the downloaded file name should be "users.csv"
Then the downloaded file should be a zip archive containing the following files named:
  | users.csv |
```

### JavaScript errors — `javascript.steps.js`

```gherkin
Then there should be no JavaScript errors
Then there should be no JavaScript warnings
Then JavaScript errors should not match "third-party-sdk"
Then print JavaScript errors
```

### Relative position — `selectors.steps.js` (named selectors required)

```gherkin
Then I see logo above main nav
Then I see footer below main content
Then I see sidebar to the left of article
Then I see avatar inside of header
Then I see modal over backdrop
Then I see hero not over header
Then I see visible submit button
Then I see email field has focus
```

### Debug — `debug.steps.js`

```gherkin
Then print current URL
Then print last response
```

## Varbase / multi-role custom steps to know

We learned a lot by experimenting and working on Varbase /
Varbase-project. The custom steps that paid off:

```gherkin
# Reads worldParameters.users[<role>]:
Given I am a logged in user with the "Content admin" user
Given I am a logged in user with the username "Content admin" user

# Body-attached micro-wait (predates BBR auto-settle — drop in new features):
And wait

# Tour / Shepherd-driven wizards:
When I click next button in tour

# Label-driven checkbox state, falls back to <label for=""> lookup:
Then I should see the "Remember me" checkbox checked
Then I should see the "Newsletter" checkbox unchecked

# Element inside container (label-or-CSS):
Then I should see the "#edit-title" element in the ".form-wrapper" field
```

Implement these in `tests/step-definitions/custom.js` (see
`/webship-js-create` for the auth-helper code).

## Critical tips

1. **Web-first first.** Prefer
   `Then "<sel>" should be visible/contain text/have value within N seconds`
   over chaining `wait for AJAX to finish`. Auto-waits, less flaky.
2. **BBR auto-settle.** Tests rarely need explicit waits between actions
   and their follow-up assertions — auto-settle covers ~95% of cases.
3. **Attribute vs text.** Use the link-by-attribute or
   element-with-attribute form for `href`, `src`, `data-*`, not
   `response should contain` (text only).
4. **Selector lookup tries** label → placeholder → name → id → css class.
5. **Named selectors win** when text is ambiguous. Register once, reuse.
6. **Iframe scope.** Always `When I switch to the iframe "..."` first;
   reset with `When I switch to the root document`.
7. **Clock mocking** stops `Date.now()` drift. Wrap time-dependent UI
   with `Given the system time is "..."`.
8. **Cucumber-js v10+.** `FORCE_COLOR=1` for colored CI logs;
   `colorsEnabled` was removed in cucumber-js 10.
9. **Auth state restore** instead of re-login per scenario (saves 1-3 s).
10. **Source of truth:** the `<category>.steps.js` files +
    `node_modules/webship-js/docs/`. Always verify phrasing before
    committing.

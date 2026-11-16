# Chapter 6: Form Testing Recipes

---

Forms are the primary way users interact with web applications. Login forms, registration forms, search boxes, checkout flows, settings panels, content editors -- they are everywhere. If your forms do not work, your application does not work.

This chapter is the most recipe-dense in the book. We cover every form element type, every interaction pattern, and every validation scenario you will encounter.

---

## 6.1 Text Input Fields

### Recipe 45: Filling Text Fields

**Problem:** Registration forms are where most suites take their first real damage. The frontend team renames `id="fname"` to `id="first-name"` during a refactor, and every test that targeted the field by CSS selector dies -- even though the form works perfectly. You want to fill fields the way a user finds them: by the label they can read. Then markup refactors stop being your problem.

**Solution:**

```gherkin
Feature: Text Input
  Scenario: Fill various text fields
    Given I am on homepage
    When I go to "/register"

    # Standard text input
    And I fill in "First Name" with "Rajab"

    # Email input
    And I fill in "Email" with "rajab@example.com"

    # Password input
    And I fill in "Password" with "SecureP@ss123"

    # Phone input
    And I fill in "Phone" with "+962-79-1234567"

    # URL input
    And I fill in "Website" with "https://natshah.com"

    And I press "Register"
    And wait for AJAX to finish
    Then I should see "Registration successful"
```

**How It Works:**

`When I fill in "First Name" with "Rajab"` resolves the field **label-first**, then falls back to id, name, and placeholder:

```
┌────────────────────────────────────────────────────────────────┐
│ Field Resolution (label-first, tried in order):                │
│                                                                │
│ 1. Label text:     <label for="fname">First Name</label>      │
│                    <input id="fname" />                        │
│                                                                │
│ 2. ID attribute:   <input id="first-name" />                  │
│                                                                │
│ 3. Name attribute: <input name="first_name" />                │
│                                                                │
│ 4. Placeholder:    <input placeholder="First Name" />         │
│                                                                │
│ The FIRST match wins. Label-first means your step targets     │
│ what the USER sees -- ids and names can change in a refactor, │
│ but the visible label rarely does.                            │
└────────────────────────────────────────────────────────────────┘
```

Label-first resolution is a deliberate design choice. The label is part of the user-facing contract of the form; the `id` is an implementation detail. When a test fills in "First Name", it documents the form the way a product owner would describe it -- and it survives the frontend refactor that renames every attribute underneath.

Two variants worth knowing. For long forms, the table form fills everything in one step:

```gherkin
When I fill in the following:
  | First Name | Rajab               |
  | Email      | rajab@example.com   |
  | Phone      | +962-79-1234567     |
```

And when a field genuinely has no usable label -- custom widgets, generated markup -- you can target a test attribute directly: `When I fill in "user-email" with "rajab@example.com" by its "data-test" attribute`.

**At scale:** Across many projects, the rule that pays for itself is: fill by label, and if there is no label, that is a bug to file -- not a CSS selector to write. Unlabeled inputs fail accessibility audits too (Chapter 15), so the test convention and the a11y requirement reinforce each other. Teams that adopt `data-test` attributes as the agreed fallback get a second win: frontend developers know exactly which attributes the test suite depends on, and code review can protect them.

### Recipe 46: Verifying Input Values

**Problem:** The edit form loads, the success toast appeared yesterday, everyone moved on -- and the Username field is quietly empty because the API response changed shape and the form no longer pre-fills. "I should see" cannot catch this: input values live in the `value` property, not in the page text, so a text assertion looks straight past an empty field. You need an assertion that reads what is actually inside the input.

**Solution:**

```gherkin
Feature: Input Value Verification
  Scenario: Verify pre-filled form data
    Given I am a logged in user with the "admin" user
    When I go to "/profile/edit"
    And wait for AJAX to finish
    Then I should see "admin" value in the "Username" input element

  Scenario: Verify saved form data
    Given I am a logged in user with the "admin" user
    When I go to "/settings"
    And I fill in "Display Name" with "Tester Demo"
    And I press "Save"
    And wait for AJAX to finish
    When I reload the page
    And wait for AJAX to finish
    Then I should see "Tester Demo" value in the "Display Name" input element
```

**How It Works:**

This step resolves the field the same label-first way as `I fill in`, then reads its `value` property -- the live state of the input, not the HTML attribute it started with. The second scenario is the pattern to memorize: **fill, save, reload, verify the value**. Without the reload you are only testing that typing works; with it you are testing that the save round-tripped through the server. There is also a selector-based, web-first variant -- `Then "#display-name" should have value "Tester Demo"` -- which polls every 100ms for up to 5 seconds, useful when the value arrives asynchronously after the page settles.

### Recipe 47: Testing Placeholder Text

**Problem:** The design team rewrites the search placeholder from "Search products..." to something less helpful, or a translation update wipes it entirely, and nobody notices because placeholders are not content anyone reviews. Here is the trap when you go to test it: a placeholder is **not page text**. It is an attribute the browser paints into an empty field, so `Then I should see` looks straight past it. You need an assertion that reads the attribute itself.

**Solution:**

```gherkin
Feature: Placeholder Text
  Scenario: Search field has helpful placeholder
    Given I am on homepage
    Then "#search" should have attribute "placeholder" with value "Search products..."
```

**How It Works:**

`should have attribute "placeholder" with value "..."` is one of webship-js's web-first assertions: it polls the element every 100ms for up to 5 seconds, so a search box that mounts late (a JavaScript-rendered header, for instance) still passes without any explicit wait. Asserting on the attribute rather than on rendered text means the test passes whether the field is empty or focused -- a text-based check would flicker depending on field state.

**Pro Tip:** Placeholder checks earn their keep on multilingual sites. The placeholder is one of the easiest strings to miss in a translation pass, and one attribute assertion per locale catches it.

### Recipe 48: Testing Character Limits

**Problem:** The database column says `VARCHAR(30)`, the form says nothing, and the first user with a long name gets a raw SQL error on a Friday afternoon. Character limits live in three places -- the HTML `maxlength`, the client-side validator, and the server -- and they drift apart over time. The only way to know which one actually fires is to push an oversized value through the form and see what comes back.

**Solution:**

```gherkin
Feature: Character Limits
  Scenario: Username has maximum length
    Given I am on homepage
    When I go to "/register"
    And I fill in "Username" with "ThisIsAVeryLongUsernameExceedingTheMaximumAllowedCharacterLimitForThisField"
    And I press "Register"
    Then I should see "Username must be 30 characters or less"

  Scenario: Bio field allows longer text
    Given I am a logged in user with the "admin" user
    When I go to "/profile/edit"
    And I fill in "Bio" with "I am a software engineer with 15 years of experience in web development, automated testing, and DevOps. I specialize in BDD and continuous integration."
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Profile updated"
```

**How It Works:**

One subtlety: if the input has an HTML `maxlength="30"`, the browser silently truncates as the step types -- the server never sees the long value, and the error message scenario will fail because no error appears. That is not a broken test; it is information. It tells you the client enforces the limit. To verify the truncation itself, follow up with `Then "#username" should have value` and the 30-character prefix. The second scenario is the often-forgotten mirror image: fields that **should** accept long text. A too-aggressive validation rule that rejects a normal-length bio is just as much a bug as a missing one.

---

## 6.2 Select Dropdowns

### Recipe 49: Selecting Options by Text

**Problem:** A backend developer reorders the country list, or swaps the option values from ISO codes to database IDs, and every test that selected options by value or by index starts choosing the wrong country -- silently. The user picks "Canada" by reading the word "Canada", and your test should too. Select by visible text, and value refactors stop reaching your suite.

**Solution:**

```gherkin
Feature: Dropdown Selection
  Scenario: Select country from dropdown
    Given I am on homepage
    When I go to "/profile/edit"
    And I select "Canada" from "Country"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Profile updated"

  Scenario: Select multiple options in sequence
    Given I am on homepage
    When I go to "/register"
    And I select "Mr." from "Title"
    And I select "Canada" from "Country"
    And I select "English" from "Preferred Language"
    And I select "UTC+3" from "Timezone"
    And I press "Register"
    And wait for AJAX to finish
    Then I should see "Registration successful"
```

**How It Works:**

`When I select "Canada" from "Country"` finds a `<select>` element labeled "Country" -- the same label-first resolution as text fields -- and selects the `<option>` with visible text "Canada".

```html
<!-- This is what the step matches: -->
<label for="country">Country</label>
<select id="country" name="country">
  <option value="">-- Select --</option>
  <option value="CA">Canada</option>    <!-- ← selected by "Canada" -->
  <option value="US">United States</option>
  <option value="UK">United Kingdom</option>
</select>
```

Selecting fires the `change` event, exactly as a real selection would -- so any JavaScript listening for it (cascading dropdowns, conditional fields) runs as normal.

For multi-selects there is a companion step: `When I additionally select "Arabic" from "Languages"` adds to the current selection instead of replacing it. The plain `I select` step clears previous choices first; `I additionally select` does not. Mixing them up is the classic reason a multi-select test ends with only the last option selected.

**At scale:** Option **labels** are also translations. On multilingual projects the visible text changes per locale while the values stay fixed -- which flips the advice: a suite that must run across language variants should pin the interface language first (or use a locale-specific examples table), or your "Germany" becomes "Deutschland" and the step finds nothing. Decide per portfolio: one canonical test locale for shared suites, with a thin per-locale layer on top for translation checks.

### Recipe 50: Testing Dynamic/AJAX Dropdowns

**Problem:** Country/city cascades are the birthplace of flaky form tests. Select "Canada", and the city dropdown takes 300ms to fetch its options -- on your laptop. In CI, under load, it takes 2 seconds, and the test that "always passed locally" picks from an empty dropdown and fails. The fix is never a sleep. It is waiting for the request that fills the dropdown to actually finish.

**Solution:**

```gherkin
Feature: Dynamic Dropdowns
  Scenario: Country dropdown loads states/provinces
    Given I am on homepage
    When I go to "/address"
    And I select "Canada" from "Country"
    And wait for AJAX to finish
    And I select "Toronto" from "City"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Address saved"
```

**How It Works:**

The key is `And wait for AJAX to finish` between selecting the country and selecting the city. When "Canada" is selected, an AJAX request fetches Canada's cities. Without the wait, the city dropdown would still be empty when the test tries to select "Toronto."

```mermaid
flowchart LR
    A["Select Canada"] --> B["AJAX request fires"]
    B -->|wait for AJAX to finish| C["Cities load"]
    C --> D["Select Toronto"]
```

Under the hood this is webship-js's instrumentation at work. Every page gets a fetch/XHR wrapper installed via `addInitScript` that maintains `window.__webshipAjaxCount` -- incremented when a request starts, decremented when it completes. The wait step polls every 50ms until that counter is zero **and** no wrapped timers are pending **and** the DOM has been quiet for at least 250ms (the `__webshipLastMutation` timestamp). So it does not just wait for the network response -- it waits for the JavaScript that *renders the options into the dropdown* to finish too. That last part is what naive "wait for response" approaches miss.

In practice you often get away without the explicit step, because webship-js runs `smartSettle(page, 1500)` after every state-changing step anyway. Keep the explicit wait for cascades regardless: a slow staging API can outlast the 1500ms AfterStep budget, and the explicit step gets the full window.

**At scale:** Cascading-dropdown timing is environment-dependent, which makes it the number-one source of "passes on project A, flakes on project B" reports when a shared suite rolls out. Because webship-js waits on the AJAX counter rather than a duration, the same feature file absorbs a 100ms dev API and a 3-second staging API without edits -- no per-project timeout tuning, no `sleep 2` creeping into the repo. If a cascade still flakes, the finding is real: the app is rendering options before its own request completes.

---

## 6.3 Checkboxes and Radio Buttons

### Recipe 51: Checking and Unchecking Checkboxes

**Problem:** Notification preferences, consent boxes, feature toggles -- checkboxes carry more legal and business weight than their size suggests. The bug pattern is sneaky: a test that *clicks* a checkbox toggles it, so if the default state ever changes, the test starts silently unchecking the thing it meant to check. You want steps that declare the end state, not the gesture.

**Solution:**

```gherkin
Feature: Checkbox Interactions
  Scenario: Check multiple checkboxes
    Given I am on homepage
    When I go to "/preferences"
    And I check "Email notifications"
    And I check "SMS alerts"
    And I check "Newsletter subscription"
    And I press "Save Preferences"
    And wait for AJAX to finish
    Then I should see "Preferences saved"

  Scenario: Uncheck a checkbox
    Given I am a logged in user with the "admin" user
    When I go to "/settings"
    And I uncheck "Show email on profile"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Settings saved"
```

**How It Works:**

`When I check "Email notifications"` resolves the checkbox by its label (the same label-first strategy as text fields) and ensures it ends up **checked** -- if it is already checked, nothing happens. `I uncheck` is the mirror image. This idempotence is the whole point: the steps state an outcome, not a click. A toggle-by-click test depends on the starting state; a check/uncheck test does not, which is exactly what you want when defaults differ between fresh installs and seeded test environments. Both steps fire the `change` event, so dependent UI (a "notification frequency" field that appears when notifications are on) reacts normally.

### Recipe 52: Verifying Checkbox State

**Problem:** A privacy-sensitive default flips during a dependency update -- "Show email on profile" ships checked when it must ship unchecked. Nobody changed that code on purpose; a form library upgrade changed how defaults are applied. Default checkbox states are invisible in code review and devastating in production, which makes them perfect candidates for an explicit assertion.

**Solution:**

```gherkin
Feature: Checkbox State Verification
  Scenario: Verify default checkbox states
    Given I am a logged in user with the "admin" user
    When I go to "/settings"
    Then the "Email notifications" checkbox is checked
    And the "SMS alerts" checkbox is checked

  Scenario: Verify checkbox state after change
    Given I am a logged in user with the "admin" user
    When I go to "/settings"
    And I uncheck "Email notifications"
    And I press "Save"
    And wait for AJAX to finish
    When I reload the page
    And wait for AJAX to finish
    # Checkbox should now be unchecked (persisted)
    Then I should see "Settings"
```

**How It Works:**

`Then the "Email notifications" checkbox is checked` reads the live `checked` property of the input -- not the HTML attribute, which only reflects the initial state. The second scenario shows the persistence pattern again (change, save, reload), and the comment marks where you would strengthen it: after the reload, assert the actual state with the checkbox step rather than settling for "the page loaded". A persistence test that only checks the page title is a navigation test wearing a costume.

**At scale:** Defaults are where a product owner's acceptance criteria and your suite meet most directly. "New users get email notifications on, SMS off" is a one-line business rule that fits in two assertion steps -- readable in the HTML report by the same PO who wrote the criterion. Across a portfolio, a short `defaults.feature` per project turns "what are the defaults supposed to be?" from an archaeology exercise into a green checkmark.

### Recipe 53: Selecting Radio Buttons

**Problem:** Shipping method, payment type, account tier -- radio groups drive price and behaviour downstream, and a misfired selection corrupts everything after it in the checkout. Radios are also fiddly to automate by hand: the visible "button" is often a styled label while the real input is hidden off-screen, so naive element clicks miss. You want a step that understands radio semantics.

**Solution:**

```gherkin
Feature: Radio Button Selection
  Scenario: Select shipping method
    Given I am on homepage
    When I go to "/checkout/shipping"
    And I select radio button "Express Shipping"
    Then the "Express Shipping" radio button is selected

  Scenario: Change radio button selection
    Given I am on homepage
    When I go to "/checkout/shipping"
    And I select radio button "Standard Shipping"
    Then the "Standard Shipping" radio button is selected

    When I select radio button "Express Shipping"
    Then the "Express Shipping" radio button is selected
    # Standard is no longer selected (only one can be selected)
```

**How It Works:**

`When I select radio button "Express Shipping"` finds the radio by its label text and selects it -- and because labels are clickable proxies for their inputs, this works even when the actual `<input type="radio">` is visually hidden behind a styled control, which is how most design systems build radios today. The second scenario tests the part of radio semantics that custom-built widgets get wrong most often: selecting one option must deselect the others. With native radios the browser guarantees it; with a JavaScript "radio group" made of divs, only your test does.

### Recipe 54: Verifying Radio Button State

**Problem:** The checkout defaults to Standard Shipping -- until a template change defaults it to Overnight and customers start paying triple without noticing the pre-selection. Default radio state is a business decision encoded in one HTML attribute, and it deserves a test with its name on it.

**Solution:**

```gherkin
Feature: Radio Button State
  Scenario: Verify default selection
    Given I am on homepage
    When I go to "/checkout/shipping"
    Then the "Standard Shipping" radio button is selected

  Scenario Outline: Verify radio options
    Given I am on homepage
    When I go to "/checkout/shipping"
    And I select radio button "<option>"
    Then the "<option>" radio button is selected

    Examples:
      | option             |
      | Standard Shipping  |
      | Express Shipping   |
      | Overnight Shipping |
```

**How It Works:**

The first scenario pins the default -- no interaction, just "load the page and check which radio the user starts on". The Scenario Outline then runs once per Examples row, each as an independent scenario with a fresh page, proving every option in the group is actually selectable. That matters more than it sounds: a typo in one option's `name` attribute quietly splits it out of the radio group, and it becomes the option that can be clicked but never stays selected. Three rows in a table is cheaper than the support ticket.

---

## 6.4 Rich Text Editors

### Recipe 55: Filling CKEditor / Rich Text Fields

**Problem:** Your first attempt at testing a CKEditor field is always the same story: `I fill in "Body"` cannot find the field, because the `<textarea>` it targets is hidden -- the editor replaced it with its own editing surface. Then someone tries keyboard-typing into the surface, and it works until the editor's autocomplete, paste filtering, or toolbar state makes it flaky. Content teams live in these editors all day; your suite needs a way in that does not fight the editor.

**Solution:**

```gherkin
Feature: Rich Text Editor
  Scenario: Create a blog post with formatted content
    Given I am a logged in user with the "editor" user
    When I go to "/admin/posts/new"
    And wait for AJAX to finish
    And I fill in "Title" with "My BDD Journey"
    And I fill the rich text editor field "Body" with "<p>Today I started learning <strong>BDD</strong> with webship-js.</p>"
    And I press "Publish"
    And wait for AJAX to finish
    Then I should see "Post published"

  Scenario: Rich text with complex HTML
    Given I am a logged in user with the "editor" user
    When I go to "/admin/posts/new"
    And wait for AJAX to finish
    And I fill in "Title" with "Feature List"
    And I fill the rich text editor field "Body" with "<h2>Features</h2><ul><li>Easy to use</li><li>Powerful automation</li><li>Beautiful reports</li></ul><p>Try it <em>today</em>!</p>"
    And I press "Publish"
    And wait for AJAX to finish
    Then I should see "Post published"
```

**How It Works:**

Rich text editors (CKEditor, TinyMCE, Quill, Froala, etc.) replace standard `<textarea>` elements with complex JavaScript-powered editing surfaces. They use either:
- `contenteditable` divs
- iframes with their own document
- Shadow DOM elements

The `I fill the rich text editor field` step handles this complexity by:
1. Finding the editor instance by its field name
2. Accessing the editor's internal API
3. Setting the content programmatically

This is more reliable than trying to type into the editor, because keyboard input in editors can behave unpredictably in automated tests -- autocorrect, paste filters, and toolbar state all interfere with simulated keystrokes, and they interfere differently across editor versions.

Setting content through the editor's own API also means the editor's change events fire, so "unsaved changes" warnings, autosave timers, and character counters all behave as if a human had typed. Those autosave timers, by the way, are exactly the kind of `setTimeout` work that `window.__webshipPendingTimers` tracks -- which is why webship-js's settle logic does not move on while the editor is still mid-autosave.

**At scale:** Rich text editors are where CMS portfolios feel version drift first. We learned this maintaining tests across dozens of Varbase sites: a CKEditor major upgrade changes the DOM completely, and every suite that poked at editor internals with CSS selectors broke at once -- while suites using the field-name step kept passing, because the step targets the field, not the editor's markup. Keep editor interaction behind this one step, and an editor upgrade becomes one framework update instead of fifty broken feature files.

### Recipe 56: Mixing Rich Text with Regular Fields

**Problem:** Real content forms are a zoo: a title field, a slug, two rich text editors, a category dropdown, a featured checkbox, and an image upload -- all on one screen, all required before "Publish" does anything. This is the form your editors fill thirty times a week, which makes it the single most valuable form to automate, and the one where each element type needs its matching step.

**Solution:**

```gherkin
Feature: Complete Content Form
  Scenario: Create a full article
    Given I am a logged in user with the "editor" user
    When I go to "/admin/articles/new"
    And wait for AJAX to finish

    # Regular text fields
    And I fill in "Title" with "Getting Started with Automated Testing"
    And I fill in "Slug" with "getting-started-automated-testing"
    And I fill in "Meta Description" with "Learn how to set up automated tests"

    # Rich text editors
    And I fill the rich text editor field "Summary" with "<p>A beginner-friendly guide to automated testing.</p>"
    And I fill the rich text editor field "Body" with "<h2>Introduction</h2><p>Automated testing saves time and catches bugs early.</p><h2>Getting Started</h2><p>Install webship-js and write your first test in minutes.</p>"

    # Dropdown
    And I select "Technology" from "Category"

    # Checkbox
    And I check "Featured Article"

    # File upload
    And I attach the file "test-image.png" to "Featured Image"

    # Submit
    And I press "Publish"
    And wait for AJAX to finish
    Then I should see "Article published successfully"
```

**How It Works:**

Each element type gets its specialist step, but they all share the same label-first resolution -- so the scenario reads like the form itself, top to bottom. The comments grouping the steps are not decoration: when this scenario fails at step 9 of 12, the report shows exactly which group broke, and a content editor can read the scenario and confirm it matches their real workflow. That review -- "yes, this is how we publish" -- is worth more than any code review for catching a missing required field. For the three plain text fields at the top, the table step (`When I fill in the following:`) would compress them to one step; with a mixed form like this, many teams prefer the explicit one-step-per-field version because the diff stays readable when the form changes.

---

## 6.5 File Uploads

### Recipe 57: Single File Upload

**Problem:** File uploads stop manual-testing habits dead: the OS file picker is a native dialog the browser cannot script, so "just click Browse and pick a file" does not translate to automation. Meanwhile the upload path -- validation, storage, thumbnail generation -- is some of the most failure-prone code in any application. You need a way to hand the browser a file without ever opening the dialog.

**Solution:**

```gherkin
Feature: File Upload
  Scenario: Upload a profile picture
    Given I am a logged in user with the "admin" user
    When I go to "/profile/edit"
    And I attach the file "test-image.png" to "profile_picture"
    And I press "Upload"
    And wait for AJAX to finish
    Then I should see "Profile picture updated"
```

**How It Works:**

`When I attach the file "test-image.png" to "profile_picture"` does the following:

1. Looks for the file at `{assets_folder}/test-image.png`
2. Finds the file input element (`<input type="file" name="profile_picture">`)
3. Uses Playwright's `setInputFiles()` to attach the file

`setInputFiles()` is the trick that makes this possible: it sets the input's file list directly at the protocol level and fires the `change` event, so the page reacts exactly as if a user had picked the file -- preview thumbnails render, upload progress starts -- but no native OS dialog ever opens. That dialog is the reason generic click automation cannot do uploads; Playwright simply routes around it.

The file must exist in your `assets_folder` (configured in `cucumber.js`). If the file is not found, the step fails with a clear error message -- before touching the browser, which makes a missing asset easy to tell apart from a broken upload.

**At scale:** Keep the test assets folder small, versioned, and shared. A handful of canonical files -- a tiny valid PNG, an oversized image, a PDF, a file with a unicode name -- covers nearly every upload scenario on every project, and committing them to the suite repository means CI runners and new laptops get identical bytes. Resist real-world-sized files: a 4MB "realistic" image uploaded by forty nightly suites is pure CI cost with zero added coverage. Test size limits with a file just over the limit, not a huge one.

### Recipe 58: Testing Upload Validation

**Problem:** Someone uploads a PDF where the avatar goes, and either the app politely refuses -- or the image pipeline chokes, the profile page renders a broken image for every visitor, and you find out from a screenshot in Slack. Upload validation is written once and then forgotten; meanwhile file-type checks are also a security boundary. The rejection paths deserve the same coverage as the happy one.

**Solution:**

```gherkin
Feature: Upload Validation
  Background:
    Given I am a logged in user with the "admin" user
    When I go to "/profile/edit"

  Scenario: Upload valid image
    When I attach the file "test-image.png" to "profile_picture"
    And I press "Upload"
    And wait for AJAX to finish
    Then I should see "Profile picture updated"

  Scenario: Upload invalid file type
    When I attach the file "sample.pdf" to "profile_picture"
    And I press "Upload"
    Then I should see "Only image files are allowed"

  Scenario: Upload with no file selected
    When I press "Upload"
    Then I should see "Please select a file"
```

**How It Works:**

Three scenarios, three different code paths: acceptance, type rejection, and the empty submit. Notice the rejection scenarios skip `wait for AJAX to finish` -- client-side validation answers instantly, and if your app validates file type on the server instead, webship-js's automatic post-step settle covers the round trip anyway. The `Background` keeps each scenario independent: each one starts freshly logged in on the edit page, so the PDF rejection cannot leak state into the valid-upload run. One more case worth adding when your app has size limits: a file just over the limit, expecting the size message -- it completes the validation triangle of type, size, and presence.

---

## 6.6 Form Submission and Validation

### Recipe 59: Submit and Verify Success

**Problem:** The contact form is often a company's single biggest lead source, and it breaks in the most boring ways -- a misconfigured mail service, a CSRF token expiry, a JavaScript error swallowing the submit. Each time, it breaks silently: the page looks fine, the user clicks Send, and nothing arrives. The Tuesday before a release, "does the contact form still work, end to end?" should be answered by a tagged scenario, not by someone sending themselves a test message.

**Solution:**

```gherkin
Feature: Contact Form
  @smoke
  Scenario: Submit a complete contact form
    Given I am on homepage
    When I go to "/contact"

    # Fill all fields
    And I fill in "Name" with "Tester Demo"
    And I fill in "Email" with "rajab@example.com"
    And I select "Technical Support" from "Department"
    And I fill in "Subject" with "Question about testing"
    And I fill in "Message" with "How do I test modals with webship-js?"

    # Submit
    And I press "Send Message"
    And wait for AJAX to finish

    # Verify success
    Then I should see "Thank you for your message"
    And I should see "We will respond within 24 hours"
    And I should not see "Error"
```

**How It Works:**

The shape is the canonical form test: fill everything, submit, then assert **three** things -- the specific success message, a second confirmation detail, and the absence of an error. That last negative assertion matters more than it looks: plenty of broken forms show the thank-you banner *and* an error box, because the banner is rendered optimistically before the request resolves. Only the trio together proves a clean submission.

The `@smoke` tag marks this as a scenario that runs on every deploy, not just nightly. A form this important earns a place in the fastest feedback loop you have.

**At scale:** Tags are the contract between QA and the pipeline -- agree on a portfolio-wide vocabulary (`@smoke` on every deploy, `@regression` nightly, `@critical` blocking release) and enforce it in review, because the day every project means the same thing by `@smoke`, your CI configuration becomes one shared template instead of forty bespoke ones. It also gives delivery managers a sentence they can use: "all @smoke scenarios green on all projects" is a release-readiness statement that needs no translation.

### Recipe 60: Testing Required Field Validation

**Problem:** A frontend rewrite ships, the form looks identical, and one `required` attribute fell off the Email field during the rebuild. Now the backend receives contact submissions with no reply address. Required-field rules are individually trivial and collectively fragile -- every form rebuild, library swap, or field rename is a chance to drop one. The empty-submit test is thirty seconds to write and catches all of them at once.

**Solution:**

```gherkin
Feature: Required Field Validation
  Scenario: Submit form with missing required fields
    Given I am on homepage
    When I go to "/contact"
    And I press "Send Message"
    Then I should see "Name is required"
    And I should see "Email is required"
    And I should see "Message is required"

  Scenario: Submit form with partial data
    Given I am on homepage
    When I go to "/contact"
    And I fill in "Name" with "Rajab"
    And I press "Send Message"
    Then I should not see "Name is required"
    And I should see "Email is required"
    And I should see "Message is required"
```

**How It Works:**

The first scenario submits a completely empty form and expects every required-field message at once -- one scenario, full coverage of the rule set. The second is the sharper instrument: fill exactly one field and assert its error *disappears* while the others *remain*. That `should not see "Name is required"` line catches a validation system that dumps every error regardless of input, and the surviving errors catch one that clears too eagerly. Together the two scenarios pin the per-field behaviour, not just "some red text appeared". No AJAX wait is needed when validation runs client-side -- the messages render synchronously on submit.

### Recipe 61: Testing Email Validation

**Problem:** Someone tightens the email regex to stop spam signups and accidentally rejects every address with a `+` tag -- including the QA team's own test accounts. Email validation has two failure directions: accepting garbage and rejecting real addresses, and the second one quietly costs you users. Testing it one scenario per address would be fifty lines of copy-paste; a Scenario Outline makes it a table.

**Solution:**

```gherkin
Feature: Email Validation
  Scenario Outline: Email format validation
    Given I am on homepage
    When I go to "/register"
    And I fill in "Email" with "<email>"
    And I press "Submit"
    Then I should see "<message>"

    Examples: Invalid emails
      | email               | message                    |
      | plaintext           | Please enter a valid email |
      | missing@tld         | Please enter a valid email |
      | @no-local.com       | Please enter a valid email |
      | user @space.com     | Please enter a valid email |
      | .start@example.com  | Please enter a valid email |

    Examples: Valid emails
      | email                     | message              |
      | user@example.com          | Registration successful |
      | user.name@example.com     | Registration successful |
      | user+tag@example.com      | Registration successful |
      | user@subdomain.example.com | Registration successful |
```

**How It Works:**

Each Examples row expands into a full, independent scenario -- fresh navigation, fresh form, one email, one expected message. The two named Examples blocks are the documentation trick: "Invalid emails" and "Valid emails" appear as group headings in the HTML report, so a product owner reviewing it sees the validation policy laid out as a table they could have written themselves. The valid block is the half teams skip -- `user+tag@example.com` is the canary for over-tight regexes, and it belongs in every email validation table.

**At scale:** Remember the multiplication: nine rows × full browser scenario × every project that inherits this feature file. Keep the table to *representative* cases -- one per failure class, not every malformed string you can imagine. Exhaustive format checking belongs in a unit test against the validator function, where a thousand cases cost milliseconds; the browser table only needs to prove the validator is wired to the form and the message reaches the user.

### Recipe 62: Testing Custom Validation Messages

**Problem:** "Invalid input" tells the user nothing; "Password must be at least 8 characters" tells them exactly what to fix. Your team agreed on helpful, specific messages -- and then a library upgrade replaced half of them with its generic defaults, and signups dipped because nobody could tell why their password was refused. The exact wording is part of the user experience. Assert it.

**Solution:**

```gherkin
Feature: Custom Validation
  Scenario: Password strength requirements
    Given I am on homepage
    When I go to "/register"
    And I fill in "Password" with "weak"
    And I press "Register"
    Then I should see "Password must be at least 8 characters"

  Scenario: Password match validation
    Given I am on homepage
    When I go to "/register"
    And I fill in "Password" with "StrongPass123!"
    And I fill in "Confirm Password" with "DifferentPass123!"
    And I press "Register"
    Then I should see "Passwords do not match"
```

**How It Works:**

Asserting the exact message text does two jobs. It verifies the validation logic fired (the rule works), and it verifies the *copy* survived (the user gets the helpful version, not a fallback). The cross-field check in the second scenario -- password versus confirmation -- is worth singling out because it is the validation type most often implemented only on the client; if your server skips it, an API user can create a mismatched account. Pair this UI scenario with an API-level check (Chapter 11) when the rule is security-relevant.

---

## 6.7 Complex Form Patterns

### Recipe 63: Multi-Step Wizard Forms

**Problem:** Wizards multiply everything that can go wrong with a form by the number of steps, then add transitions on top. A user abandons registration at step 2 of 3, and analytics cannot tell you whether the form was too long or simply broken. When a wizard breaks, it usually breaks *between* steps -- the Next button saves but does not advance, or advances without saving -- and only a test that walks the whole path catches it.

**Solution:**

```gherkin
Feature: Registration Wizard
  As a new user
  I want to complete a step-by-step registration
  So that I can set up my account properly

  Scenario: Complete multi-step registration
    Given I am on homepage
    When I go to "/register"

    # Step 1: Account Details
    Then I should see "Step 1 of 3"
    And I should see "Account Details"
    When I fill in "Email" with "new@example.com"
    And I fill in "Password" with "SecurePass123!"
    And I fill in "Confirm Password" with "SecurePass123!"
    And I press "Next"
    And wait for AJAX to finish

    # Step 2: Personal Information
    Then I should see "Step 2 of 3"
    And I should see "Personal Information"
    When I fill in "First Name" with "Rajab"
    And I fill in "Last Name" with "Natshah"
    And I select "Canada" from "Country"
    And I press "Next"
    And wait for AJAX to finish

    # Step 3: Preferences
    Then I should see "Step 3 of 3"
    And I should see "Preferences"
    When I check "Email notifications"
    And I select "English" from "Language"
    And I check "I agree to the Terms of Service"
    And I press "Create Account"
    And wait for AJAX to finish

    # Confirmation
    Then I should see "Welcome, Rajab!"
    And I should see "Your account has been created"
```

**How It Works:**

The pattern is **assert, act, assert**: confirm you are on the step you think you are on (`Then I should see "Step 1 of 3"`) *before* filling anything, then fill, advance, and confirm the next step. Without those position assertions, a wizard that silently fails to advance produces the worst kind of failure -- the test types step 2's data into step 1's fields and reports an error three steps away from the actual bug. Each `wait for AJAX to finish` covers the step transition, which on most wizards is an AJAX save; webship-js's AJAX counter ensures the next assertion only runs once the new step has actually rendered.

The final assertion -- "Welcome, Rajab!" -- closes the loop: data entered at step 2 surfaces after step 3, proving the wizard carried state across all transitions, not just displayed three screens.

**At scale:** One long scenario per wizard is the right call -- resist splitting it into per-step scenarios, because steps 2 and 3 cannot be reached without doing step 1 anyway, and three scenarios that each replay the prefix cost triple the runtime for the same coverage. The wizard is a single user behaviour: getting registered. Tag it `@critical` and let it block releases; a broken registration wizard is a stop-everything bug on any project.

### Recipe 64: Testing Wizard Back Navigation

**Problem:** A user gets to step 3 of registration, realizes they typoed their email, clicks Previous -- and step 1 is blank. They close the tab. Losing a user's typed data on back-navigation is one of the most rage-inducing bugs a form can have, and it never shows up in forward-only testing, which is the only testing wizards usually get.

**Solution:**

```gherkin
Feature: Wizard Navigation
  Scenario: Go back preserves data
    Given I am on homepage
    When I go to "/register"

    # Fill Step 1
    And I fill in "Email" with "test@example.com"
    And I press "Next"
    And wait for AJAX to finish

    # Fill Step 2
    And I fill in "First Name" with "Rajab"
    And I press "Next"
    And wait for AJAX to finish

    # Go back to Step 2
    When I press "Previous"
    And wait for AJAX to finish
    Then I should see "Step 2 of 3"
    # First Name should still be filled
    Then I should see "Rajab" value in the "First Name" input element

    # Go back to Step 1
    When I press "Previous"
    And wait for AJAX to finish
    Then I should see "Step 1 of 3"
    # Email should still be filled
    Then I should see "test@example.com" value in the "Email" input element
```

**How It Works:**

The crucial assertions here are the **value** checks, not the step indicators. "Step 2 of 3" appearing proves navigation works; `"Rajab" value in the "First Name" input element` proves the wizard re-hydrated the field from wherever it stores in-progress state. Those are separate mechanisms, and the second one is the one that breaks. The test walks back through *every* step it visited, because wizards commonly persist the most recent step's data and lose the earlier ones -- a partial bug that a single Previous click would miss.

### Recipe 65: Conditional Form Fields

**Problem:** Select "Business" and three company fields are supposed to appear; select "Personal" and they must vanish -- along with their `required` flags, or personal users get blocked by validation on fields they cannot see. Conditional field logic is rewritten every time the form library changes, and its failure mode is invisible: the form renders, looks complete, and is wrong. Both directions need a test, because show-but-never-hide is the most common way this logic decays.

**Solution:**

```gherkin
Feature: Conditional Fields
  Scenario: Company fields appear for business accounts
    Given I am on homepage
    When I go to "/register"
    And I select "Business" from "Account Type"
    And wait for AJAX to finish
    Then I should see "Company Name"
    And I should see "Tax ID"
    And I should see "Company Address"

  Scenario: Company fields hidden for personal accounts
    Given I am on homepage
    When I go to "/register"
    And I select "Personal" from "Account Type"
    And wait for AJAX to finish
    Then I should not see "Company Name"
    And I should not see "Tax ID"
```

**How It Works:**

The `change` event fired by `I select` triggers the show/hide logic, and the settle that follows gives the DOM time to finish mutating -- webship-js's mutation observer (`window.__webshipLastMutation`) keeps the wait alive until the DOM has been quiet for 250ms, which covers both instant toggles and animated reveals. The negative assertions in the second scenario are doing the real work: "Personal account does not ask for a Tax ID" is a business rule, and `I should not see` is how a business rule about absence becomes executable. If the fields are hidden with CSS rather than removed from the DOM, the selector-level web-first form -- `Then "#tax-id" should be hidden` -- makes the same assertion with surgical precision.

**At scale:** Conditional logic is exactly the kind of acceptance criterion product owners write naturally -- "if the user picks Business, ask for company details" -- and these two scenarios are that sentence, verbatim, in executable form. Get POs to write conditional-field rules as Given/When/Then in the ticket itself; on teams that do this, QA stops reverse-engineering intent from mockups and the scenario lands in the suite the same sprint the feature ships.

### Recipe 66: Auto-Complete/Suggestion Fields

**Problem:** Auto-complete fields are a two-part timing puzzle: the app waits for you to stop typing (a debounce timer), then fires a search request, then renders suggestions. A test that types and immediately looks for "Laptop" loses the race on slow CI runners every time -- the classic test that passes locally and flakes in the pipeline.

**Solution:**

```gherkin
Feature: Auto-Complete
  Scenario: Select from auto-complete suggestions
    Given I am on homepage
    When I go to "/search"
    And I fill in "Search" with "lap"
    And wait for AJAX to finish
    Then I should see "Laptop"
    And I should see "Laptop Stand"
    When I click "Laptop"
    And wait for AJAX to finish
    Then I should see "Laptop Details"
```

**How It Works:**

This recipe is where webship-js's timer tracking earns its keep. The debounce in an auto-complete is a `setTimeout` -- and webship-js wraps `setTimeout` via `addInitScript`, counting pending timers in `window.__webshipPendingTimers`. The settle logic refuses to declare the page quiet while a timer is pending **or** an AJAX request is in flight **or** the DOM is still mutating. So the wait covers the whole chain -- debounce, fetch, suggestion render -- in one step, with no magic number that matches your laptop but not your CI runner. Then `When I click "Laptop"` selects the suggestion exactly as a user would: by clicking its visible text.

**Pro Tip:** Type a *prefix* ("lap"), not the full term. Asserting that two different suggestions appear ("Laptop" and "Laptop Stand") proves the suggestion engine is matching, not just echoing your input back.

---

## 6.8 Form Recipes Quick Reference

```
┌──────────────────────────────────┬──────────────────────────────────┐
│ Form Action                      │ webship-js Step                  │
├──────────────────────────────────┼──────────────────────────────────┤
│ Fill text input                  │ I fill in "Field" with "value"   │
│ Fill many fields at once         │ I fill in the following: (table) │
│ Select dropdown option           │ I select "Option" from "Field"   │
│ Add to a multi-select            │ I additionally select "Option"   │
│                                  │   from "Field"                   │
│ Check a checkbox                 │ I check "Label"                  │
│ Uncheck a checkbox               │ I uncheck "Label"                │
│ Select radio button              │ I select radio button "Label"    │
│ Upload a file                    │ I attach the file "f" to "Field" │
│ Fill rich text editor            │ I fill the rich text editor      │
│                                  │   field "Name" with "HTML"       │
│ Click a button (submit)          │ I press "Button Text"            │
│ Click a link                     │ I click "Link Text"              │
│ Verify text visible              │ I should see "text"              │
│ Verify text NOT visible          │ I should not see "text"          │
│ Verify checkbox checked          │ the "X" checkbox is checked      │
│ Verify radio selected            │ the "X" radio button is selected │
│ Verify input value               │ I should see "X" value in the    │
│                                  │   "Y" input element              │
└──────────────────────────────────┴──────────────────────────────────┘
```

---

## Chapter Summary

- webship-js covers all standard form elements: text, select, checkbox, radio, file, rich text
- The `buildSelector` helper tries multiple strategies to find elements (label, placeholder, name, id)
- Use `wait for AJAX to finish` after actions that trigger server requests
- Scenario Outline is perfect for validation testing with many inputs
- Test the complete form workflow: fill → submit → verify
- Complex patterns (wizards, conditional fields) just combine basic steps
- Rich text editors need the special `fill the rich text editor field` step
- File uploads require test files in the `assets_folder`

Next chapter: mastering the art of waiting for dynamic content.

# Chapter 43: Advanced Field Types

---

A text input is easy. `fill in "X" with "Y"` covers it. The interesting fields are the ones that don't behave like text. A color picker has a hex value and a paint swatch. A datetime field has two halves. A multi-value field accepts a list of strings, one chip per value. A WYSIWYG editor is an entire embedded document with its own keystrokes. Picking these up correctly is the difference between a test suite that covers real forms and one that pretends every input is a `<input type="text">`.

Webship-js ~2.0 adds dedicated steps for each of these field types. This chapter walks them.

---

## Field-Type Overview

| Field type | HTML | webship-js step |
|------------|------|-----------------|
| Color | `<input type="color">` | `fill in the color field "X" with the value "Y"` |
| Datetime | `<input type="datetime-local">` | `fill in the datetime field "X" with date "Y" and time "Z"` |
| Datetime range | start + end pair | `fill in the start datetime field ...` + `fill in the end datetime field ...` |
| Multi-value (chips) | tag-style input | `fill in the multi-value field "X" with the following values:` |
| WYSIWYG | Trix, TinyMCE, ProseMirror, etc. | `fill in the WYSIWYG field "X" with the "Y"` |
| Text selection | any text field | `select "X" text in "Y" field` |

---

## Recipe 419: Filling a Color Picker

### Problem
White-label products live or die on the brand settings page. A tenant sets their primary color, presses Save -- and if that save quietly fails, their whole product wears a stranger's identity, which a client notices faster than any functional bug. Manual testers skip this field for an honest reason: the native color-picker popup is a fiddly OS dialog that's miserable to operate by hand. Which is exactly the argument for automating it.

### Solution

```gherkin
Feature: Brand settings

  Scenario: Save brand color
    Given I am on "/settings/brand"
    When I fill in the color field "Primary color" with the value "#358FC8"
    And I press "Save"
    Then ".color-saved-toast" should be visible within 2 seconds

  Scenario: Color value persists
    Given I am on "/settings/brand"
    Then the color field "Primary color" should have the value "#358FC8"
```

The step never opens that OS color dialog -- it couldn't, reliably: the picker UI is platform-native, different on every OS, and absent in headless runs. Instead it writes the value straight into the `<input type="color">` and fires the input events a framework binding listens for. Hex values in three- and six-digit forms both work (`#fff`, `#358FC8`); browsers normalise the input's value to lowercase six-digit internally, so the assertion compares the normalised form and the matcher is case-insensitive -- `#358FC8` and `#358fc8` are the same color and the same pass. The two-scenario split is the real design here: every scenario gets a fresh browser context, so when the second scenario reads the value back, nothing could have survived from the first except through the server. A green second scenario *is* the persistence proof.

**At scale:** in a multi-tenant portfolio, brand settings are per-client configuration and this same set-save-reload triple applies to every tenant theme. Tag it `@branding`, drive the target site per pipeline through `LAUNCH_URL`, and keep the expected hex values in per-environment data rather than hardcoded in twenty copies of the feature. The run that catches a broken brand save the week before a client's launch pays for itself in one phone call you never receive.

---

## Recipe 420: Filling a Single Datetime Field

### Problem
The first attempt at this test was a plain fill-in with `"06/15/2026 2:30 PM"` -- and the field ended up empty, the form complained about a missing start time, and someone lost an hour to it. That's the datetime-local trap: the input *displays* dates in the user's locale, but it only *accepts* the ISO wire format, and a plain text fill speaks to the display. The booking flow can't be tested at all until this one field cooperates, so it blocks every scenario behind it.

### Solution

```gherkin
  Scenario: Set reservation start
    Given I am on "/booking/new"
    When I fill in the datetime field "Start" with date "2026-06-15" and time "14:30"
    And I press "Continue"
    Then ".chosen-start" should have text "Jun 15, 2026 2:30 PM" within 2 seconds
```

The step composes the two parts into the ISO wire value (`2026-06-15T14:30`) and sets it on the native input in one call -- date in ISO (`YYYY-MM-DD`), time in 24-hour (`HH:MM`). Your test speaks the wire format; the browser renders whatever the user's locale dictates; the two never collide. The field itself is found label-first -- "Start" matches the label, then falls back through id, name, and placeholder -- the same resolution order as every webship-js fill step. Note what the assertion checks: not the input's raw value but the *rendered* confirmation, "Jun 15, 2026 2:30 PM". That closes the whole round trip -- ISO in, application formatting out -- in one line.

**At scale:** locale is a classic source of "passes on my machine, fails in CI" for date tests, because a developer's laptop renders `15/06/2026` while the CI image renders `06/15/2026`. This step makes the *input* side locale-proof -- ISO is ISO everywhere. For the assertion side, either pin the CI image's locale and document it in the shared pipeline config, or assert on values you control (the input's ISO value) instead of locale-formatted display strings. Decide once for the portfolio; per-project workarounds for locale are how forty suites end up with forty date formats.

---

## Recipe 421: Filling Only the Date Part

### Problem
Business rule: every booking ends at 18:00, so the form pre-fills the time and users only pick a day. If your test fills both halves anyway, it silently *overwrites* the pre-filled time -- and now the test edits the very state it was supposed to observe. The day the prefill rule regresses and the field comes up blank, the both-halves test stays green, because it papered over the hole itself. Tests that touch only what the user touches catch what the user would catch.

### Solution

```gherkin
  Scenario: Set only the date
    Given I am on "/booking/new"
    When I fill in the date part of the datetime field "Start" with "2026-06-15"
    Then "input[name='start']" should have value "2026-06-15T18:00"
```

The "date part" variant updates only the date component of the input's value and leaves the time component exactly as it found it; a "time part" variant covers the inverse case. That makes the assertion quietly powerful: `2026-06-15T18:00` -- the ISO serialisation of a datetime-local value -- verifies *two* facts in one string. The date half is what your step set; the `18:00` half is what the business rule was supposed to pre-fill, untouched by the test. If the prefill breaks, this scenario fails on the time half even though every step in it succeeded. One assertion, your action and their rule, checked together.

**Pro Tip:** when a form mixes user-entered and rule-driven values, write the assertion against the combined result like this rather than asserting the halves separately -- the combined string also catches the subtle bug where setting one half *clears* the other.

---

## Recipe 422: Datetime Ranges

### Problem
Check-in and check-out are one logical field wearing two inputs, grouped under a single "Reservation" label. Naive label resolution finds two candidates and picks one -- and which one it picks is an accident of DOM order, which means the test fills check-in twice and the booking spans negative three nights. The duration between the two fields is also the number the guest pays by, so this isn't a corner to leave untested until a date-math refactor gets it wrong.

### Solution

```gherkin
  Scenario: Set check-in and check-out
    Given I am on "/booking/new"
    When I fill in the start datetime field "Reservation" with date "2026-06-15" and time "14:00"
    And I fill in the end datetime field "Reservation" with date "2026-06-18" and time "11:00"
    And I press "Continue"
    Then ".duration" should have text "3 nights" within 1 second
```

The start/end variants encode the convention that two datetime inputs grouped under a shared label form a range: both steps locate the pair through the "Reservation" label, then "start" addresses the first input of the pair and "end" the second. The ambiguity that breaks the plain step is resolved by saying which half you mean, in words a product owner can read aloud. The assertion is deliberately not about the inputs -- it's about `.duration`, the *derived* value. June 15 to June 18 is three nights only if the application's date math is right, so "3 nights" tests the computation, which is the part of a range field that actually carries money. Each fill is a state-changing step, so the AfterStep `smartSettle` lets any recalculation finish before Continue fires.

**Pro Tip:** add one more scenario with end before start and assert the form refuses it. Range fields have two contracts -- compute the span, and reject the impossible span -- and the second one is the one that reaches production untested.

---

## Recipe 423: Multi-Value Chip Fields

### Problem
Every project picks a different tag-input library, and every library commits a chip differently: one on Enter, one on comma, one on blur. The naive approach -- fill the text box with `"testing playwright cucumber-js"` -- produces zero chips, or one giant chip, depending on the library's mood, and the article saves with broken taxonomy that nobody notices until the tag archive pages come up empty. The new hire writing their first content test hits this wall on day one.

### Solution

```gherkin
  Scenario: Add three tags
    Given I am on "/articles/new"
    When I fill in the multi-value field "Tags" with the following values:
      | testing      |
      | playwright   |
      | cucumber-js  |
    Then ".tag" should have a count of 3 within 1 second
    And ".tag" should contain text "playwright"
```

The step works the field like a user does, one value at a time: type the value, commit it (Enter first, falling back to comma for libraries that separate on commas), then *wait for the chip to render before starting the next value*. That wait is the part you'd get wrong by hand -- chip libraries re-render the input after each commit, and typing into an element mid-re-render is how characters silently vanish. The table rows are an ordered list, entered top to bottom. The assertions then split the proof: the web-first `count of 3` says the right *number* of chips materialised (catching the all-one-chip failure), and `contain text` spot-checks that a value survived intact rather than being split mid-word.

**At scale:** chip widgets vary more across a portfolio than any other field type -- one product on Select2, one on a homegrown React chip box, one on whatever the CMS ships. One step phrasing over all of them keeps feature files identical from project to project, so a QA engineer moving between suites never relearns "how do we add tags here." When a project's library is too exotic for the Enter/comma convention, wrap the difference in a project-local custom step *with the same wording* -- the vocabulary is the asset worth protecting.

---

## Recipe 424: Filling a WYSIWYG Editor

### Problem
The body field isn't an input. It's an entire embedded editor with its own document model, and depending on the project it's Trix, TinyMCE, or ProseMirror -- each hiding the real text somewhere different. A plain fill-in finds either nothing or a hidden textarea the editor ignores, and here's why that hurts more than one broken test: every content scenario in the suite needs body text before it can test anything else. This one step gates dozens of tests, so when it's flaky, the whole suite has a reputation problem.

### Solution

```gherkin
  Scenario: Write an article
    Given I am on "/articles/new"
    When I fill in "title" with "Hello World"
    And I fill in the WYSIWYG field "Body" with the "This is the first paragraph of the article."
    And I press "Save"
    Then ".article-body" should contain text "first paragraph" within 3 seconds
```

The step detects which editor it's talking to by inspecting the DOM around the labelled field, then speaks that editor's dialect: for ProseMirror it sets the editable region's text, for Trix it dispatches the events Trix listens for, and for TinyMCE it writes inside the editor's iframe -- because TinyMCE is literally a page within your page. That detection is what lets one phrasing serve every project. The assertion deliberately checks the *rendered article* after Save, not the editor's internal state: content has to survive the editor's serialisation, the save request, and the re-render to count as written. A 3-second budget covers the round trip; the AfterStep `smartSettle` has already drained the save's AJAX before the assertion starts polling.

**At scale:** this is the pattern that carried Varbase testing at Vardot -- dozens of Drupal sites, every body field a rich-text editor, one step phrasing in every suite. When the editor upgraded across the distribution, the suites reported exactly which sites' content forms broke, and nobody maintained a per-project editor step to find out. If your portfolio mixes editor libraries, the single phrasing is what keeps a QA engineer's knowledge portable between projects -- the editor is an implementation detail, and the feature files stop caring about it.

---

## Recipe 425: Selecting Text Inside a Field

### Problem
To test bold, you first need a selection -- and a selection is invisible state with no element to click and no attribute to read. The human gesture is click-drag across exactly one word; scripting that with raw mouse coordinates means measuring where "Hello" is painted, which breaks on the next font change, zoom level, or viewport. Teams skip formatting tests for exactly this reason, and then a toolbar refactor ships bold-that-doesn't-bold to a newsroom.

### Solution

```gherkin
  Scenario: Bold a single word
    Given I am on "/articles/new"
    When I fill in the WYSIWYG field "Body" with the "Hello world"
    And I select "Hello" text in "Body" field
    And I press the keys "Control+B" on the element ".ProseMirror"
    Then ".ProseMirror strong" should have text "Hello" within 1 second
```

`select X text in Y field` finds the first occurrence of the substring within the field's value and sets the selection range to it -- by text, not by coordinates, so layout, fonts, and viewport size never enter the picture. That selection becomes the target for whatever keyboard shortcut comes next; the chord is scoped onto the editor element (Recipe 418's trick) so it bubbles from inside the editor's keymap. The assertion closes on the markup: a `strong` element containing exactly "Hello" -- not "Hello world", which would mean the selection grabbed too much. Selection, chord, markup: three steps that together pin down a behaviour most suites silently leave untested.

**Pro Tip:** "first occurrence" is load-bearing. If the body says "Hello, Hello, goodbye", the step selects the first "Hello" -- when you need a later one, make the search string longer and unambiguous ("Hello, goodbye") rather than hoping.

---

## Recipe 426: Selecting a Character Range

### Problem
Sometimes the test is about position, not words: a copy-paste flow, a masked input that reveals a slice, a "select the middle and retype" edge case a user reported. There is no substring to search for because the *position* is the point -- and dragging a mouse across exactly nine characters is not something automation should ever attempt. You need to say "characters 5 through 13" and have it mean that, on every machine, at every font size.

### Solution

```gherkin
  Scenario: Select character range
    Given I am on "/profile"
    When I fill in "bio" with "I am a developer working on test tooling."
    And I select from 5 to 14 text in "bio" field
    And I press the keys "Control+C"
    Then the clipboard should contain "a develop"
```

The range form uses zero-indexed character positions, half-open: `from 5 to 14` selects indices 5 through 13 -- here, "a develop". Half-open intervals look pedantic until you notice they match how JavaScript slices strings: `value.substring(5, 14)` returns exactly the selected text, so you can compute your expected value in a console with the same numbers the step uses, no off-by-one negotiation. The chord then copies the selection and the clipboard assertion verifies what actually landed there -- which tests the full path users rely on, selection through clipboard, not just that a highlight appeared.

**Pro Tip:** pin the field's value inside the scenario, as this one does with its fill step, instead of relying on fixture data. Index-based selections are only as stable as the string under them -- a fixture edit that shifts one character turns "a develop" into nonsense and the failure into a puzzle.

---

## Recipe 427: Selecting All Text

### Problem
A pre-filled field is a different animal from an empty one. Rename and change-email flows are where masked inputs and controlled components fight back: typing appends to the old value instead of replacing it, or the component restores the old value on blur because it never registered a proper clear. The user's actual gesture is select-all, delete, type -- and when a field misbehaves, the most reliable fix is to make the test perform exactly that gesture instead of arguing with the component's change detection.

### Solution

```gherkin
  Scenario: Clear by select-all and type
    Given I am on "/profile"
    Then "input[name=display-name]" should have value "Old Name"
    When I select all text in "display-name" field
    And I press the key "Delete"
    And I fill in the field "display-name" with "New Name"
    Then "input[name=display-name]" should have value "New Name"
```

`select all text in X field` is the keyboard equivalent of clicking into the field and pressing Ctrl+A -- the whole value becomes the selection, and the `Delete` keypress collapses it to a genuinely empty field, with every keystroke flowing through the events a framework's change tracking listens to. Programmatic clears sometimes skip those events, which is precisely how a component "restores" a value the test thought it removed. Two details carry weight here. The opening assertion pins the precondition (`Old Name` is really there), so when fixture data changes, the test fails at the setup line with an honest message instead of at the end with a baffling one. And the closing assertion reads the input's value -- a check that the replacement took, not merely that typing happened.

---

## Recipe 428: Field State Assertions

### Problem
Modern signup forms enable the Submit button off field validity -- which makes "the email field is valid" load-bearing UI state that no screenshot can show you. The classic regression: someone tightens the input's validation pattern for an edge case, every address with a plus sign in it turns invalid, and signups sag for a week while the form looks perfectly normal in every demo. The validity state changed; nothing visible did. That gap is exactly what these assertions watch.

### Solution

```gherkin
  Scenario: Email field validates
    Given I am on "/signup"
    Then the field "email" should be required
    And the field "email" should be empty
    When I fill in "email" with "user@example.com"
    Then the field "email" should not be empty
    And the field "email" should have "valid" state
```

The state predicates wrap HTML5 constraint validation -- the browser's own verdict on the field, the same machinery that decides whether a form submits. `required` reads the field's constraint; `empty`/`not empty` read its value; `valid`/`invalid` ask the element's validity state, which accounts for every constraint at once (type, pattern, required, length). The remaining states -- `pristine`, `dirty`, `touched`, `untouched` -- describe interaction history: whether the value has changed and whether the user has visited the field, the signals form libraries use to decide *when* to show an error. The scenario's shape is before-and-after on purpose: required and empty before typing, non-empty and valid after, so a failure tells you which transition broke rather than just "validation is off."

**At scale:** make field-state scenarios a portfolio convention -- every project's signup and checkout forms get one -- so reviewers stop guessing whether client-side validation has coverage. There's a bonus audience: these assertions read like acceptance criteria verbatim. "Email is required and must be valid before submit enables" is a sentence a product owner wrote in the ticket and a line a QA engineer can point to in the green run -- the rare case where the criteria and the test literally match, which is exactly the sign-off conversation you want.

---

## Field-Type Pitfalls

**Native vs custom date pickers:** A custom calendar component is not an `<input type="date">`. It's usually a button that opens a popover with day cells. For those, use `click "15"` in the popover -- the datetime steps in this chapter target the native HTML5 inputs only.

**Multi-value field separators:** Some chip libraries treat space as a separator; others wait for Enter. The webship-js step tries Enter first, falls back to comma. For exotic libraries, dispatch the key event manually.

**WYSIWYG content escaping:** When the editor body contains HTML, the `fill in the WYSIWYG field` step sets the plain text -- HTML tags are typed literally, not parsed. To set HTML content, use a custom step that calls the editor's API.

**Color picker on Safari/Linux:** The native color picker UI varies by platform. webship-js bypasses the UI and writes the value directly to the input -- consistent across browsers.

Advanced field types reward precision. Use the right step for the right widget, and your form tests stay readable as the forms get richer.

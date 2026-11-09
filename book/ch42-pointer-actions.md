# Chapter 42: Pointer Actions, Drag and Drop, Keyboard

---

A `click` covers maybe sixty percent of user input. The other forty are pointer actions that look simple on the surface but require precise sequencing: drag, drop, hover, right-click, middle-click, double-click, tap, key combos. A reordering UI uses drag-and-drop. A context menu opens on right-click. A "copy to clipboard" button needs Ctrl+C to be a credible test. A mobile gesture is a `tap`, not a click.

Webship-js ~2.0 adds first-class steps for every pointer and keyboard action a real user can produce. This chapter walks each one.

---

## Pointer Actions Map

<!-- diagram: page-lifecycle -->

| User gesture | webship-js step |
|--------------|-----------------|
| Single click | `When I click "X"` |
| Double click | `When I double-click on "X"` |
| Right click | `When I right-click on "X"` |
| Middle click | `When I middle-click on "X"` |
| Hover | `When I hover over "X"` |
| Tap (mobile) | `When I tap on "X"` |
| Drag and drop | `When I drag "A" to "B"` |
| Click with modifier | `When I click on "X" while holding "Shift"` |
| Press key | `When I press the key "Enter"` |
| Press chord | `When I press the keys "Control+S"` |

---

## Recipe 409: Double-Click to Edit Inline

### Problem
Your spreadsheet-style grid enters edit mode on double-click -- and the regression that broke it shipped clean through manual QA, because every tester single-clicked, saw the selection highlight, and moved on. Edit mode is the kind of interaction nobody re-checks once it works. Then an event-handler refactor replaces the `dblclick` listener with two `click` handlers and a home-made timer, and editing turns into a coin flip on slower machines. The support tickets say "sometimes I can't type in cells", which is the least debuggable sentence in the language.

### Solution

```gherkin
Feature: Inline editing

  Scenario: Double-click cell enters edit mode
    Given I am on "/sheets/abc"
    When I double-click on "td[data-cell='A1']"
    Then "td[data-cell='A1'] input" should be focused within 1 second
    When I fill in the field "A1" with "100"
    And I press the key "Enter"
    Then "td[data-cell='A1']" should have text "100" within 1 second
```

The double-click fires the platform's native `dblclick` event -- two down/up pairs inside the OS double-click interval, then the `dblclick` itself -- not two independent click steps, which the browser would treat as two separate selections. Apps that listen for `dblclick` (spreadsheets, tree views, map zoom) need this step rather than `click click`. The rest of the scenario rides webship-js's wait machinery: `should be focused` is a web-first assertion that polls every 100ms, so the editor's mount animation costs nothing; and because the double-click is a state-changing step, the AfterStep hook runs `smartSettle` afterwards, draining any AJAX or pending timers before the fill-in starts. The fill never races the editor mounting. `Enter` commits the value, and the final text assertion proves the cell re-rendered with it -- the full edit round-trip, not just the toggle.

**Pro Tip:** assert focus on the *input inside the cell*, as here, not on the cell itself. "Edit mode" is really "an editable element exists and owns focus" -- testing the CSS class the cell gains is testing the paint, not the behaviour.

---

## Recipe 410: Right-Click for Context Menu

### Problem
Context menus are invisible features. No button advertises them, so manual testers forget they exist, analytics undercounts them, and your file manager's right-click menu -- the one carrying "Rename" and "Delete" -- can die in a refactor without anyone noticing. The classic killer is a well-meaning global click handler that swallows the `contextmenu` event for some overlay logic. Users don't file bugs for a menu they don't know they lost; they just call support and say the product "feels broken."

### Solution

```gherkin
  Scenario: Context menu opens on right-click
    Given I am on "/files"
    When I right-click on "li[data-file='report.pdf']"
    Then ".context-menu" should be visible within 1 second
    When I click "Rename"
    Then "input[name=newName]" should be focused within 1 second
```

The step fires a real `contextmenu` event sequence -- right-button down, up, then `contextmenu` -- so a page handler that calls `event.preventDefault()` and renders its own menu behaves exactly as in production. One worry you can cross off: the *browser's* native context menu never renders during automation, so there is nothing to dismiss and nothing to screenshot over your UI. From there the scenario is ordinary webship-js: a web-first visibility poll for the menu, a click on "Rename" by its visible text, and a focus assertion proving the rename input is ready for typing -- which is the real user promise, not merely "a menu appeared."

**Pro Tip:** add a second scenario that right-clicks, then clicks elsewhere, then asserts `".context-menu" should be hidden`. Menus that open are half the feature; menus that refuse to close are the half users actually rage about.

---

## Recipe 411: Middle-Click to Open in New Tab

### Problem
Power users read search results the same way everywhere: middle-click five of them, then work through the tabs. That convention only survives if your results are real anchors. The day a frontend "improvement" turns the link into a `div` with a click handler and router navigation, middle-click dies -- and the complaint arrives phrased as "your site broke my mouse," usually from your most engaged users. You want a test that proves the link still behaves like a link.

### Solution

```gherkin
  Scenario: Middle-click opens in new tab
    Given I start recording network requests
    And I am on "/search?q=playwright"
    When I middle-click on "a.search-result__link"
    Then a GET request to "**/articles/playwright-101" should have been made
```

Middle-click is browser convention for "open in new tab/window." The step fires the native auxiliary-button event; webship-js doesn't switch tabs automatically -- if you need to interact with the new tab, see the tab-switching family in Chapter 12. That's why this scenario asserts on the *network* instead of the new tab: `I start recording network requests` arms the request log before anything loads, and the final step checks that the middle-click actually produced a navigation request for the article. The `**/` glob matches the full URL whatever host it ran against, so the same scenario passes on local, staging, and production via `LAUNCH_URL`. The ordering matters -- recording starts first, for the same reason the download steps in Chapter 40 arm their listener before clicking: assert-after-the-fact only works if someone was listening when the fact happened.

---

## Recipe 412: Hover to Reveal Tooltip

### Problem
The help text for the trickiest field on the form lives in exactly one place: a tooltip. Tooltips break in the most boring ways -- a z-index change buries them under the header, a CSS refactor renames the class that positions them, a library upgrade changes the trigger event -- and manual passes never catch it, because the testers already know what the field means and never hover. The user who actually needed the hint gets nothing, and the form's abandon rate quietly climbs.

### Solution

```gherkin
  Scenario: Help tooltip appears on hover
    Given I am on "/profile"
    Then "#help-tooltip" should be hidden
    When I hover over "#help-icon"
    Then "#help-tooltip" should be visible within 500 milliseconds
    And "#help-tooltip" should contain text "Your username must be unique"
```

`hover over` moves the pointer to the element's centre and leaves it there -- CSS `:hover` rules and JavaScript `mouseenter`/`mouseover` handlers both fire, exactly as they would for a real pointer. Notice the scenario asserts `hidden` *first*: that proves the hover caused the reveal, rather than the tooltip having been visible (or stuck open) all along. The tightened `within 500 milliseconds` budget is deliberate. The web-first assertion polls every 100ms, and a tooltip behind a 300ms CSS transition fits comfortably -- but the default 5 seconds would also pass a tooltip that takes four seconds to appear, which is its own UX bug. Timeout budgets encode expectations; spend them like money. The final `contain text` line is the content regression test -- remember, this tooltip is the only home that sentence has.

**At scale:** tooltips almost always come from one design-system component shared across every product in the portfolio. Test the component's *mechanics* -- reveal, hide, positioning -- deeply in one suite, the design system's own; in the other nineteen suites, assert only the content each product puts inside it. When the design system upgrades and the interaction breaks, one suite goes red with a precise answer instead of twenty suites going red with the same noise.

---

## Recipe 413: Tap Gesture on Mobile

### Problem
On the mobile build, the dismiss control listens for `touchstart` -- not `click` -- because the team wanted the snappiest possible response. Your desktop-viewport suite passes every night, and meanwhile the notification cannot be dismissed on an actual phone. This is the trap with touch UIs: a `click` step fires mouse events, and an element listening only to touch events never hears them, so the one flow your mobile users hit hardest is precisely the one your suite never exercises. The bug report eventually arrives as a screen recording from someone's commute.

### Solution

```gherkin
  Scenario: Notification dismisses on tap
    Given I am viewing the site on a "mobile" device
    And I am on "/"
    Then ".notification" should be visible
    When I tap on ".notification__dismiss"
    Then ".notification" should be hidden within 1 second
```

`tap` fires `touchstart` and `touchend` -- the genuine touch sequence -- rather than a synthesised mouse click, which is exactly what a UI listening to touch events directly needs to hear. The device step before it matters just as much: it gives the page a phone-sized viewport with touch support, the context in which a tap makes sense at all. The assertion pair tells the dismissal story -- visible before, hidden within a 1-second budget after, enough headroom for a slide-out animation without tolerating a hang.

**At scale:** give touch-dependent scenarios their own CI lane. Tag them `@mobile` in every project and run them in a dedicated job where the device emulation is configured once, centrally -- one place to decide which device profile the whole portfolio tests against. Without that convention, mobile coverage decays into whatever each project's author happened to emulate the week they wrote the test, and "works on mobile" means something different in every repo.

---

## Recipe 414: Drag and Drop Between Lists

### Problem
The kanban board *is* the product: drag a card to "Done" and the API records the move. It is also the interaction most likely to break in a frontend upgrade, because there are two incompatible ways to implement dragging -- native HTML5 drag-and-drop and pointer-event libraries -- and a library swap silently changes which one your page speaks. The cruel version arrives the Tuesday before release: the new board animates beautifully, the card glides across the screen, and the drop never persists. Visually perfect, functionally void, and only a test that checks *state*, not motion, will catch it.

### Solution

```gherkin
  Scenario: Card moves to Done column
    Given I am on "/board/sprint-12"
    When I drag "[data-card='TASK-42']" to "[data-column='done']"
    Then "[data-column='done'] [data-card='TASK-42']" should be visible within 2 seconds
    And "[data-column='todo'] [data-card='TASK-42']" should not be attached
```

The drag-and-drop step is HTML5-DnD aware: it fires `dragstart`, `dragenter`, `dragover`, `dragleave`, `drop`, and `dragend` in the sequence a real drag produces. For libraries that never touch native DnD and listen to pointer events instead (dnd-kit and friends), the step still works -- it dispatches pointer events as a fallback, which is exactly the both-implementations insurance the library-swap problem calls for. The assertions are the both-sides pattern: the Done column *gained* the card and the To-do column *lost* it -- `not be attached` checks actual DOM removal, a stronger claim than `hidden`. The 2-second budget absorbs the drop handler's animation and save call; and since the drag is a state-changing step, the AfterStep `smartSettle` drains the save's XHR (watching `window.__webshipAjaxCount` fall to zero) before the first assertion even starts polling.

**At scale:** the selectors here are the real portfolio decision. Agree once -- across every project -- that draggables carry `data-card`-style identity attributes and targets carry `data-column`, then put the names in each project's shared selectors JSON under `tests/selectors/` so feature files reference names while the raw selectors live in one file. When the next board library changes the DOM, you update one JSON file per project, not forty feature files, and the convention survives team turnover because it is written down where the tests load it.

---

## Recipe 415: Click With a Modifier Key

### Problem
Shift-click range selection is muscle memory from every file manager since the nineties, and your bulk-actions feature depends on it. It has regressed before: a selection-state refactor rebuilt the list on every click and dropped the anchor item, so Shift-clicking item three selected exactly one row. Manual QA missed it completely -- clicking three items one at a time also works, and that is how testers click. Only a test that holds the modifier the way a user does can tell the two behaviours apart.

### Solution

```gherkin
  Scenario: Shift-click extends selection
    Given I am on "/library"
    When I click on "li[data-book='1']"
    And I click on "li[data-book='3']" while holding "Shift"
    Then ".selected-book" should have a count of 3 within 1 second
```

The `while holding` clause presses the modifier down, performs the click, and releases -- so the page's click handler sees `event.shiftKey === true`, indistinguishable from the real gesture. Supported modifiers: `Shift`, `Control`, `Alt`, and `Meta` (Cmd on macOS); the key is held only for that one click, so the next step starts unmodified. The count assertion is the part that actually proves range semantics: items 1 and 3 were clicked, but the assertion demands *three* selected elements -- item 2 must have been swept up by the range. `should have a count of` is web-first, polling until the selection class lands on all three or the budget runs out. Asserting "the thing I didn't click is selected" is the whole test; the two clicks are just setup.

---

## Recipe 416: Pressing a Single Key

### Problem
Your docs site copied the `/` shortcut from GitHub: press slash anywhere, land in the search box. Then a navbar refactor re-scopes the global keydown listener, the shortcut dies, and no dashboard ever notices -- shortcut users don't file bugs, they grumble, reach for the mouse, and quietly stop being your fastest users. Keyboard shortcuts are features with no pixels of their own, which makes them the easiest features to break without knowing.

### Solution

```gherkin
  Scenario: Slash opens search
    Given I am on "/docs"
    When I press the key "/"
    Then "#search-input" should be focused within 1 second
    And "#search-input" should be visible
```

`press the key` dispatches a single keydown/keyup pair to the page with nothing focused -- and that "nothing focused" is the point of the test. A global shortcut's contract is precisely that it works when the user is just *on the page*, so the keystroke must be caught by a document-level listener, not by a lucky focused element. The two assertions then close the loop: `focused` proves focus actually moved into the search input (the user can type immediately -- the real promise of a focus shortcut), and `visible` proves the input isn't focused-but-hidden, a genuinely confusing state that focus management bugs produce more often than you'd think. For component-scoped shortcuts, add `on the element "..."` to target the keystroke -- that variant is Recipe 418's territory.

**At scale:** keep a one-feature-file shortcut inventory per product -- every binding, one scenario each, tagged `@shortcuts`. It costs minutes and it doubles as living documentation: when a product owner asks "what shortcuts do we actually support?", the feature file is the answer, and unlike the wiki page it runs every night. Shortcuts are also an accessibility surface, so your a11y review can point at the same file.

---

## Recipe 417: Pressing a Key Combination

### Problem
Ctrl+S is a promise: the user's work will not vanish. Your editor wires it to a save endpoint -- but one missing `event.preventDefault()` after a refactor hands the keystroke back to the browser, which helpfully offers to save the *web page* as HTML. The user dismisses the dialog, assumes their document saved, and closes the tab. Nothing in the UI looked broken; the data is simply gone. You want proof, on every run, that the chord reaches your handler and produces a write.

### Solution

```gherkin
  Scenario: Ctrl+S saves the document
    Given I start recording network requests
    And I am on "/docs/42/edit"
    When I fill in the field "title" with "New title"
    And I press the keys "Control+S"
    Then a POST request to "**/api/docs/42/save" should have been made
```

The keys step accepts a `+`-separated chord: `Control+S`, `Shift+Tab`, `Control+Alt+Delete`. The modifiers are pressed and held in order, the final key is pressed and released, then the modifiers release -- the same event stream a human produces. The interesting choice is the assertion: the proof of saving lives on the wire, not in the UI. `I start recording network requests` arms the request log before the page loads, and the final step demands a POST matching the `**/api/docs/42/save` glob -- which matches the full URL on any host, so the scenario travels across environments with `LAUNCH_URL`. Asserting the POST instead of a "Saved!" toast tests the contract (data left the browser) rather than the decoration; toasts have a way of staying green after the request beneath them broke.

**At scale:** chords are where "passes locally, fails in CI" gets personal. Your CI farm runs Linux and tests `Control+S`; half the team develops on macOS where the same muscle memory is `Meta+S`. Decide once, portfolio-wide, which chord the suites assert -- the CI platform's -- and write the decision down next to the pipeline config, or wrap the difference in one shared custom step that branches on `process.platform`. The expensive outcome is each project solving it differently, so every new hire rediscovers the mismatch one red pipeline at a time.

---

## Recipe 418: Pressing Keys on a Specific Element

### Problem
Ctrl+B must bold text only when the editor owns focus -- the same chord at document level should do nothing. Scoping bugs here are maddening because they pass in development by accident: the editor *happens* to have focus when you try it, so the shortcut works. Then in production a toast notification steals focus for two seconds, the user hits Ctrl+B during that window, the keystroke lands on the document, and bold "doesn't work sometimes." Intermittent, focus-dependent, unreproducible on demand -- the worst kind of ticket.

### Solution

```gherkin
  Scenario: Bold shortcut bolds selected text
    Given I am on "/articles/new"
    When I fill in the WYSIWYG field "content" with the "Hello world"
    And I select "Hello" text in "content" field
    And I press the keys "Control+B" on the element ".ProseMirror"
    Then ".ProseMirror strong" should have text "Hello" within 1 second
```

The `on the element` variant delivers the key events to that specific node, so the keydown originates *inside* the editor and bubbles up through it -- which is what editor keymaps (ProseMirror plugins, TinyMCE shortcuts) actually listen for. Without the scope, the keystroke originates at the document, never passes through the editor's handler, and the test would fail even though the feature works -- or worse, pass for the wrong reason. The preceding select-text step (its mechanics live in Chapter 43) sets up the selection the chord operates on; the final assertion checks the *semantic result* -- a `strong` element containing exactly "Hello" -- rather than a toolbar button's active state. Markup is the contract a bold command makes; the glowing button is just its reflection.

---

## Drag and Drop, the Robust Way

Drag-and-drop is the flakiest of all pointer actions. A few rules:

1. **Always assert before and after.** Don't assume the drag worked -- check that the target list got the item AND the source list lost it.
2. **Allow a budget.** Drop handlers often run animations or save calls. Use `within N seconds` on the assertion.
3. **Use stable selectors.** `nth-child` selectors break the moment a drag changes order. Prefer `data-id` attributes that survive reordering.

```gherkin
# Bad
When I drag "li:first-child" to "li:last-child"

# Good
When I drag "[data-card='TASK-42']" to "[data-column='done']"
```

The bad version targets positions; the good version targets identity.

---

## Common Gotchas

**Hover and CSS transitions:** A `:hover` style with a 300ms transition needs `within 500 ms` on the assertion, not the default 5 seconds.

**Tap vs click on mobile:** If you test the mobile site with `mobile` viewport but still use `click`, you'll fire mouse events instead of touch events. Pages that listen specifically for `touchstart` won't respond. Use `tap` for mobile-only flows.

**Modifier keys on Mac vs Linux:** `Control+S` saves on Linux/Windows, `Meta+S` saves on macOS. If your tests run cross-platform, branch on `process.platform`.

**Drag-and-drop in iframes:** The drag must start and end in the same frame. To drag from the parent into an iframe (or vice versa), simulate the gesture with separate pointer-down, pointer-move, and pointer-up steps -- a single `drag X to Y` won't cross frame boundaries.

Pointer and keyboard actions are where a test suite goes from "covers the happy path" to "covers what users actually do." Pick the step that matches the user's gesture, not the one that's closest to "click."

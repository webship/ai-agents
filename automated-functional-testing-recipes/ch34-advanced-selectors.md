# Chapter 34: Advanced Selectors -- Named Locators, Spatial Relations, Viewports

---

Let me paint a picture. Your feature files are readable today. `When I click "Login"` tells any stakeholder what happens. Then the designer rebuilds the login button into a custom component that no longer accepts a visible label. Now you need `nav.primary a[data-role='auth-cta']:not(.disabled)`. One CSS selector, buried in one test, unreadable to anyone who doesn't read the DOM.

Multiply by 50 tests. Then multiply by the eight other projects your team maintains, each with its own copy of that selector, each drifting at its own pace. The readability you started with is gone -- and so is any hope of fixing a markup change in one place.

Webship-js ~2.0 solves this with a **named selector system**. You register long CSS or XPath strings once with a friendly name, then refer to them in tests by that name. You also get spatial assertions (`above`, `below`, `inside`, `over`), viewport breakpoints by name (`xs`, `md`, `xl`), and focus / text-selection steps that work on any labeled field.

The whole family lives in `webship-selectors.js`. It is loaded automatically in the default `cucumber.js`.

---

## The Named Selector System

<!-- diagram: named-selectors -->

```
┌────────────────────────────────────────────────────────────────┐
│ Registry (per scenario)                                        │
│                                                                │
│  CSS selectors:                                                │
│   "page header"  → "header.page-header"                        │
│   "main nav"     → "nav[role='navigation']"                    │
│   "cta button"   → ".cta .btn-primary"                         │
│                                                                │
│  XPath selectors:                                              │
│   "page title"   → "//h1[contains(@class,'page-title')]"       │
│   "admin menu"   → "//nav[contains(@class,'admin-menu')]"      │
│                                                                │
│  Resolution order:                                             │
│   1. css registry   (first lookup)                             │
│   2. xpath registry (fallback -- wrapped with "xpath=")          │
│                                                                │
│  Sources:                                                      │
│   - worldParameters.selectors.css / .xpath                     │
│   - JSON files listed in worldParameters.selectors.files       │
│   - Inline "When I add ... selector" at runtime                │
│   - Bulk "Given I define css selectors:" via data tables       │
└────────────────────────────────────────────────────────────────┘
```

Every scenario starts with a fresh copy of the registry. A `Before` hook rebuilds it from `worldParameters.selectors` -- the inline `css` and `xpath` maps first, then every JSON file listed in `files` (read from `filesPath`, which defaults to `./tests/selectors/`), in order. Later entries override earlier ones with the same name -- **last wins**. This lets each scenario tweak selectors without leaking to the next.

The same `selectors` block carries two more settings you will meet later in this chapter: a scroll offset (60px by default, tunable with `WEBSHIP_SELECTORS_OFFSET` -- headroom for sticky headers when elements get scrolled into position) and the named viewport breakpoints (`WEBSHIP_SELECTORS_BREAKPOINTS`).

---

## Recipe 310: Registering a Named Selector Inline

### Problem
The front-end team ships a redesign on Thursday. The "Add to cart" button is now an icon-only custom component -- no visible label, no stable text for a click-by-name step to find. The quick fix is pasting `.product__buy button[type=submit]` straight into the feature file, and that is the moment your Gherkin stops being readable by anyone outside the QA team. You need the raw CSS. You just don't want it living in the scenario.

### Solution

```gherkin
Feature: Cart test with named selectors

  Scenario: Add to cart
    Given I am on "/products/101"
    When I add "add-to-cart" selector for ".product__buy button[type=submit]" css selector
    And I click add-to-cart
    Then I see visible add-to-cart
```

The step writes `add-to-cart` into the scenario's CSS registry -- the same per-scenario map the `Before` hook seeds from `worldParameters` and your JSON files. Registration lasts for this scenario only; the next scenario gets a fresh copy, so nothing leaks between tests. Every later step that takes a name (`I click ...`, `I see visible ...`, the spatial assertions) resolves it through the registry: CSS map first, XPath map as the fallback. Register the same name twice and the last registration wins -- handy when one scenario needs to shadow a shared name without touching the shared file. When the CSS changes, you edit one line.

### XPath variant

```gherkin
Scenario: Click via XPath
  Given I am on "/dashboard"
  When I add "admin link" selector for "//a[contains(normalize-space(.),'Admin')]" xpath selector
  And I click admin link
```

The step automatically prefixes `xpath=` when routing to Playwright's locator API, so the expression is evaluated as XPath instead of CSS. Same registry rules apply: scenario-scoped, last wins. One note on resolution order -- if the same name exists in both registries, the CSS entry wins, because lookup always tries the CSS map first.

---

## Recipe 311: Defining Selectors in Bulk

### Problem
Your layout scenario needs ten selectors, and the inline `add` step works -- once per selector. Now the preamble is longer than the test, and the reviewer on your PR scrolls past ten lines of plumbing to find the three assertions that actually matter. The new hire copies that pattern into the next feature, and the noise compounds from there.

### Solution

```gherkin
Scenario: Landing page layout
  Given I am on homepage
  And I define css selectors:
    | topbar  | .topbar                |
    | header  | header.site            |
    | nav     | nav[role=navigation]   |
    | main    | main                   |
    | footer  | footer                 |
    | cta     | .hero .cta-primary     |
  Then I see visible topbar, header, nav, main, footer
  And I see header above main
  And I see main above footer
```

The `define css selectors:` step takes a Gherkin data table. Each row is a `name | selector` pair that lands in the same per-scenario registry as the inline `add` step -- one write per row, last write wins, so a duplicate name later in the table (or in a later step) overrides the earlier one. For distinct names, order does not matter. There is a side benefit nobody plans for: the table reads like a glossary. A reviewer can sanity-check the names against the design system without parsing a single CSS rule.

### XPath bulk variant

```gherkin
Given I define xpath selectors:
  | first heading | //h1[1]                            |
  | login link    | //a[contains(normalize-space(.),'Login')] |
  | admin menu    | //nav[contains(@class,'admin-menu')]      |
```

---

## Recipe 312: Loading Selectors from a JSON File

### Problem
The same `.hero-banner` selector is now pasted into fourteen scenarios across six feature files. Then the redesign lands and the class becomes `.hero-banner--v2`. It is the Tuesday before release, you are grepping feature files, and you fix eleven of the fourteen -- the suite goes red in three places nobody remembers owning. Selectors that are shared in spirit but duplicated in practice always end this way.

### Solution

Create `tests/selectors/homepage.json`:

```json
{
  "css": {
    "hero":      ".hero-banner",
    "hero cta":  ".hero-banner .btn-primary",
    "nav":       "nav[role='navigation']",
    "footer":    "footer.site-footer"
  },
  "xpath": {
    "page title":  "//h1[contains(@class,'page-title')]",
    "breadcrumb":  "//nav[@aria-label='breadcrumb']"
  }
}
```

Configure `cucumber.js`:

```js
worldParameters: {
  selectors: {
    filesPath: './tests/selectors/',
    files: ['homepage.json'],
  }
}
```

Every scenario auto-loads the file at start. Reference names directly:

```gherkin
Scenario: Homepage smoke
  Given I am on homepage
  Then I see visible hero, nav, footer
  And I see hero above footer
```

### Load a file inline
If you need a file only for one scenario:

```gherkin
When I add selectors from "admin-panel.json" file
Then I see visible admin menu
```

The file path is relative to `worldParameters.selectors.filesPath`.

### Layering rule
Load order: worldParameters defaults → files listed in worldParameters → inline `add` / `define` steps. Later entries override earlier ones. You can load multiple files to share common selectors across areas.

Under the hood this is a plain merge. The `Before` hook copies the `css` and `xpath` maps from `worldParameters`, then reads each file in `files` from `filesPath` in the order listed and merges its `"css"` and `"xpath"` keys over the top. Both keys are optional -- a file can be CSS-only. Two guardrails worth knowing: only `.json` files are accepted (anything else throws a clear error instead of half-loading), and a file that fails to parse fails the scenario immediately, with the file path in the message. You find out at the first scenario, not at assertion forty.

**At scale:** this file is where multi-project teams win. Keep one `shared.json` per design system -- not per project -- and distribute it (a git submodule, an npm package, or a plain CI copy step all work). Twenty projects built on the same theme then share one set of names: `hero` means the same thing in every suite, failure messages read identically across projects, and a markup change in the design system is one pull request to one file instead of twenty. The selector file becomes the contract between QA and front-end -- front-end edits the values, QA owns the names.

---

## Recipe 313: Asserting Visibility with Named Selectors

### Problem
Post-deploy smoke check, Friday at 5pm. The hero rendered but the CTA inside it did not -- and your assertion stops at the first missing element. You fix that, rerun, and discover the headline is missing too. Every discovery costs a full CI round trip. You want one step that checks the whole section and tells you everything that is broken, the first time.

### Solution

```gherkin
Scenario: Hero section fully rendered
  Given I am on homepage
  And I define css selectors:
    | hero    | .hero-banner       |
    | headline| .hero-banner h1    |
    | cta     | .hero-banner .btn  |
  Then I see visible hero, headline, cta
```

Each name resolves through the registry, then the step runs Playwright's native auto-retrying `locator.waitFor({ state: 'visible' })` -- up to 5 seconds per selector. That is polling, not a snapshot: a component that hydrates late still passes, with no static sleep anywhere. And by the time a `Then` step runs, webship-js's AfterStep hook has already run `smartSettle`, so in-flight fetches and pending timers from the previous step are drained -- you are asserting against a quiet page. If any name fails, the step keeps going, collects every failure, and throws one error listing all of them. You see everything broken, not just the first.

**At scale:** a one-line visibility contract is the cheapest smoke test you can run across a portfolio. Give every project the same five names from the shared selector file, tag the scenario `@smoke`, and the nightly run answers "did every site render its critical sections?" in one pass -- with failure messages that read identically from project to project. That sameness is what makes a cross-project dashboard worth looking at.

### Negation

```gherkin
Then I don't see modal, overlay, cookie banner
```

Uses `waitFor({ state: 'hidden' })`. In Playwright terms, "hidden" also covers "not in the DOM at all" -- so the same step passes for a modal that closed and for one that never rendered. Good for checking a closed modal stays closed.

---

## Recipe 314: Spatial Relation Assertions

### Problem
A front-end refactor swaps the header from flexbox to grid. Every element still exists, every visibility check passes, and the screenshot diff is just noisy enough that someone approves it. On the demo call, the nav is sitting under the logo. Presence checks cannot catch this -- nothing disappeared. You need assertions about *where things are*: logo left of nav, footer below main, modal on top of the page.

### Solution

```gherkin
Scenario: Layout contract
  Given I am on homepage
  And I define css selectors:
    | header  | header                |
    | logo    | header .site-logo     |
    | nav     | header nav            |
    | main    | main                  |
    | footer  | footer                |
    | modal   | .modal.is-open        |
  Then I see logo to the left of nav
  And I see header above main
  And I see main above footer
  And I see nav inside of header
  And I see modal over main
```

Each assertion measures real geometry. The step scrolls the element into view if needed (`scrollIntoViewIfNeeded`, with a short grace period), reads its `boundingBox()`, then adds `window.scrollX` / `window.scrollY` -- because `boundingBox()` returns viewport coordinates, and you want absolute page coordinates that stay comparable even when two elements are a full screen apart. For `over` / `not over` it also reads the computed `z-index` through `evaluate()`, since Playwright has no native API for computed styles. If either element has no box at all (hidden or detached), the step fails with a message naming which one.

### Full list of spatial relations

| Relation                  | Passes when                                                |
|---------------------------|------------------------------------------------------------|
| `A above B`               | B.top >= A.top + A.height                                   |
| `A below B`               | A.top >= B.top + B.height                                   |
| `A to the left of B`      | A.right <= B.left                                           |
| `A to the right of B`     | A.left >= B.right                                           |
| `A inside of B`           | A's box fully contained in B's box                          |
| `A outside of B`          | A's box fully contains B's box                              |
| `A over B`                | A and B overlap AND A has lower (or equal) z-index than B   |
| `A not over B`            | A and B do not overlap OR z-indexes do not satisfy `over`   |

### Multi-subject dispatch

```gherkin
Then I see heading, subline above list, footer
```

This runs the Cartesian product: `heading above list`, `heading above footer`, `subline above list`, `subline above footer`. All four must pass. Like the visibility steps, the dispatcher checks every pair before failing and reports all broken pairs in one error -- four assertions, one CI round trip.

### Why spatial assertions matter
Visual regression catches pixel drift. Spatial assertions catch structural drift. A designer swaps `display: flex` for `display: grid` and the nav ends up below the logo instead of beside it. A pixel diff might still match (the elements look similar). A `logo to the left of nav` assertion fails instantly.

**At scale:** when twenty sites share one base theme, write the layout contract once -- header above main, logo left of nav, footer last -- against shared selector names, and run it on every site. A theme regression then fails everywhere at once with the same message, which tells you immediately it is the theme and not one project's override. That distinction is exactly what a delivery manager needs the morning a "layout is broken" ticket lands: one red site means a local change, twenty red sites means roll back the theme.

---

## Recipe 315: Viewport Breakpoints by Name

### Problem
A "broken on mobile" bug report lands, and nobody can answer the obvious question: which widths do our tests actually cover? One suite hard-codes 375 x 667, another picked 414 x 896, a third has a typo -- 786 instead of 768 -- that has been silently testing the wrong breakpoint for months. Hand-typed viewport numbers drift, and drifted numbers are worse than none, because they look like coverage.

### Solution

Webship-js ships named breakpoints out of the box:

| Name | Width | Height | Target                        |
|------|-------|--------|-------------------------------|
| xs   | 375   | 667    | phone portrait                |
| sm   | 576   | 800    | large phone / phablet         |
| md   | 768   | 1024   | tablet portrait               |
| lg   | 992   | 768    | small laptop / tablet landscape |
| xl   | 1200  | 900    | desktop (default)             |
| xxl  | 1400  | 900    | wide desktop / HD             |
| xxxl | 1920  | 1080   | Full HD / large monitor       |

Use them in Gherkin:

```gherkin
Scenario: Mobile layout
  Given I am viewing the site on a xs screen
  And I am on homepage
  Then I see visible hero
  And I see nav to the right of logo

Scenario: Desktop layout
  Given I am viewing the site on a xl screen
  And I am on homepage
  Then I see nav to the right of logo
  And I see cta inside of hero
```

### Quoted form for CI clarity

```gherkin
Given I am viewing the site on a "xxl" screen
```

Both work. Quote when the name could be mistaken for a word. The step also accepts `device` in place of `screen` -- `viewing the site on a "md" device` reads better in some scenarios.

### Custom breakpoints

```js
worldParameters: {
  selectors: {
    breakpoints: {
      phone:  { width: 360, height: 640  },
      tablet: { width: 900, height: 1280 },
      hd:     { width: 1920, height: 1080, default: true },
    }
  }
}
```

Now `Given I am viewing the site on a phone screen` works. Override via env:

```bash
export WEBSHIP_SELECTORS_BREAKPOINTS='{"phone":{"width":360,"height":640}}'
```

Two mechanics worth knowing. First, the step works because webship-js ships `viewport: null` in `playwright.config.ts` -- no fixed window size -- so when the step resolves the name and calls `page.setViewportSize()`, it is the single authority over the window. Ask for a name that is not defined and you get a friendly error pointing at `worldParameters.selectors.breakpoints`, not a silent fallback. Second, the env var **replaces** the whole breakpoint map -- it is parsed as one JSON object, not merged. The export above leaves you with only `phone` defined; include every breakpoint your features mention.

**At scale:** stop letting each project pick its own numbers. Define the breakpoint map once, export `WEBSHIP_SELECTORS_BREAKPOINTS` at the CI level, and every suite in the organization tests identical widths without touching a single `cucumber.js`. The payoff is an answer your QA lead can give in one sentence: "every project is tested at 375, 768, and 1200." Device-lab debates end, and a "broken on mobile" report maps to a breakpoint name the whole team shares.

---

## Recipe 316: Clicking by Named Selector

### Problem
The design system replaced your text CTA with an icon button. There is no visible label for `When I click "Login"` to match, the accessible name is an a11y backlog ticket that will not ship this sprint, and the test still has to click the thing today. Pasting the CSS into the step works -- and produces exactly the unreadable Gherkin this chapter exists to prevent.

### Solution

```gherkin
Scenario: Click CTA
  Given I am on homepage
  And I define css selectors:
    | cta | .hero .btn-primary |
  When I click cta
```

The `click <name>` step resolves the name from the registry (CSS map first, XPath second), then calls `locator.click()` -- which buys you Playwright's full actionability pipeline: auto-scroll into view, visible, stable, enabled, not covered by another element. After the click, webship-js's AfterStep hook runs `smartSettle(page, 1500)`, draining in-flight fetch/XHR calls and pending timers before your next step executes -- so whatever the click kicked off has landed before you assert on it. No wrappers, no manual waits.

### Click multiple names

```gherkin
When I click cta, secondary link
```

Runs each in sequence. If any fails, the step collects errors and reports them together.

### Combined with XPath registration

```gherkin
When I add "admin link" selector for "//a[contains(normalize-space(.),'Admin')]" xpath selector
And I click admin link
```

### Gotcha
The step pattern is liberal: `When I click <anything>`. If the name you give does not exist in the registry, you get an error telling you to define it. It will not fall back to Playwright's text matching -- for that, use the classic `When I click "Login"` step.

---

## Recipe 317: Focus and Text Selection

### Problem
A screen-reader user reports that focus lands in the wrong field after the page loads, and support cannot reproduce it -- their screenshots never show where focus is. The accessibility audit is due Thursday. You need tests that drive keyboard-first flows and pin down the exact state: this field focused, these characters selected -- the way a power user or an assistive technology actually experiences the form.

### Solution

```gherkin
Scenario: Keyboard input
  Given I am on "/register"
  When I move focus to "Email" field
  Then I see Email has focus

  When I fill in "Body" with "Release 2.0 notes"
  And I select from 0 to 7 text in "Body" field

  When I fill in "Title" with "original"
  And I select "original" text in "Title" field
```

Steps:

- `move focus to "<field>" field` -- `focus()` on the field (by label, placeholder, name, or id).
- `<name> has focus` -- asserts the element is `document.activeElement`.
- `select all text in "<field>" field` -- `selectText()` on the field.
- `select from <N> to <M> text in "<field>" field` -- `setSelectionRange(N, M)`.
- `select "<text>" text in "<field>" field` -- finds the substring, ranges the selection.

### Why this matters
Accessibility teams care about keyboard order. Power users care about tab flow. Customer-support screenshots often happen with a field focused -- these steps let you reproduce the exact state.

### Field resolution order
The steps try, in order:

1. `getByLabel(field)`
2. `getByPlaceholder(field)`
3. `getByRole('textbox', { name: field })`
4. `[name="field"]` or `#field`

First match wins. Accessibility-first lookup, matching Playwright's recommended pattern.

One implementation detail makes this more reliable than it sounds: the chain is built with Playwright's `.or()` combinator into a *single* locator, not four sequential lookups. The whole chain auto-retries together -- a label that appears 800ms after navigation still resolves, because the locator keeps polling instead of failing on the first pass. The focus assertion is equally patient: `has focus` waits for the element to attach, then polls `document.activeElement` for up to 5 seconds, so focus moved by JavaScript shortly after load still passes without a sleep. The range-selection steps run `setSelectionRange()` inside the element itself, and the substring variant reads the field's current value first -- if the text is not there, you get an error naming the missing substring, not a silent no-op.

---

## Recipe 318: Debugging the Selector Registry

### Problem
The new hire's first PR fails with "unknown selector 'admin menu'". They have three JSON files, a `worldParameters` block, and two inline `define` tables in front of them -- and no idea which layer was supposed to provide the name, or which layer overwrote it. Guessing by reading config files is slow. You want to see what the scenario actually has registered, at the moment it runs.

### Solution

```gherkin
Scenario: Debug
  Given I am on "/admin"
  When I add selectors from "admin-selectors.json" file
  Then I print css selectors
  And I print xpath selectors
```

Console output:

```
--- CSS Selectors ---
  "admin menu": nav.admin-menu
  "admin user": .admin-user-link

--- XPath Selectors ---
  "first admin link": //nav[@class='admin-menu']//a[1]
```

### When to use it
The print steps dump the live per-scenario registry -- the post-merge truth after the `Before` hook has layered `worldParameters`, then each file in `files` in listed order, then every runtime `add` / `define` step executed so far. Place the print *after* the loading step you suspect, and read the output against that order: if the value is wrong, the last layer that defines the name is your culprit; if the name is missing entirely, no layer ever defined it -- check the file name and the `filesPath` it resolves against. Once per problem. Remove the print steps before committing -- they clutter output.

**At scale:** put "print the registry, read the merge order" in your triage runbook. When one team maintains suites for many projects, the senior engineer cannot personally debug every unknown-selector failure -- a runbook line plus this step turns the same investigation into a five-minute self-serve task on any project.

---

## Recipe 319: Layering Strategy for a Large Project

### Problem
One hundred selectors, five teams, one shared file -- and on Friday afternoon someone renames `site header` to unblock their own test. Forty scenarios across two other products go red, and the weekend on-call gets paged for a "regression" that is really a naming dispute. Without a layering convention, the selector registry becomes a commons that every team grazes and nobody tends.

### Solution

Pick a layering convention:

**Layer 1: Shared project selectors** -- committed in `tests/selectors/shared.json`. Every feature gets them via `worldParameters.selectors.files`.

```json
{
  "css": {
    "site header":  "header.site-header",
    "primary nav":  "nav[role='navigation']",
    "site footer":  "footer.site-footer"
  }
}
```

**Layer 2: Feature-area selectors** -- loaded per area.

```gherkin
Feature: Admin dashboard
  Background:
    When I add selectors from "admin.json" file
```

**Layer 3: Scenario overrides** -- inline when something is unique to one test.

```gherkin
Scenario: Cart totals
  When I add "cart total" selector for ".cart-summary .total-amount" css selector
  Then I see visible cart total
```

### Benefits
- Shared layer -- central truth, edited rarely.
- Area layer -- owned by feature teams, keeps blast radius small.
- Scenario layer -- scratch work that lives with the test.

Name collisions are resolved by last-write: scenario beats file beats worldParameters. That is the intended order. It is the merge mechanics from Recipe 312 doing governance work -- the most local intent always wins, and an override is visible right there in the scenario that needs it, instead of hidden in a config file three directories away.

**At scale:** treat the shared layer like an API. Names are the interface; CSS values are the implementation. Front-end may change values freely -- but renaming or deleting a *name* requires a deprecation window where the old name keeps working (two names, one selector) and a heads-up to every consuming team. Version the shared file and distribute it like any dependency, so a project can pin it during a release freeze. This is the pattern we converged on while maintaining suites across dozens of Varbase sites at Vardot: the day selector names became a reviewed contract instead of a free-for-all was the day cross-project breakage stopped being a weekly event.

---

## Recipe 320: Selectors with Responsive Tests

### Problem
Mobile regressions keep reaching users because the suite only ever tests the desktop width. The team's first fix was copy-paste: three copies of the homepage scenario with different viewport numbers. Within a month the copies drifted -- someone updated the desktop one and forgot the others -- and the `xs` copy quietly stopped being maintained. You want one scenario that runs at every breakpoint, so the coverage cannot drift.

### Solution

```gherkin
Feature: Responsive homepage

  Background:
    Given I define css selectors:
      | header  | header              |
      | logo    | header .logo        |
      | nav     | header nav          |
      | hero    | .hero-section       |
      | cta     | .hero-section .btn  |

  Scenario Outline: Layout at <bp>
    Given I am viewing the site on a <bp> screen
    And I am on homepage
    Then I see visible header, logo, nav, hero, cta
    And I see logo to the left of nav
    And I see header above hero

    Examples:
      | bp  |
      | xs  |
      | md  |
      | xl  |
      | xxl |
```

One feature, four breakpoints. Each `Examples` row runs as an independent scenario: a fresh selector registry from the `Before` hook, a `setViewportSize()` call to the named breakpoint, and spatial geometry measured anew at that width -- so `logo to the left of nav` is genuinely re-verified at 375px, not assumed from the desktop run. The scenario name carries the breakpoint (`Layout at xs`), so the report tells you exactly which width broke before you read a single stack trace.

**At scale:** do the matrix math before you copy this pattern everywhere. Four breakpoints times thirty scenarios times twenty projects is 2,400 scenario runs a night -- real CI minutes, real money. The pattern that holds up: run `xs` and `xl` on every pull request, and the full breakpoint matrix nightly under a `@responsive` tag. Then pass `--metadata Breakpoints=xs,md,xl,xxl` to the report generator so the cross-project dashboard shows what each run actually covered. That metadata is the difference between "tests passed" and "mobile is covered" -- which is the sentence your delivery manager actually needs.

### What this catches
A designer hides the nav on mobile. Your `logo to the left of nav` assertion fails on `xs`. You get a precise error: "nav is not visible at 375 x 667". You fix the mobile layout, not the test.

---

## Putting It All Together

```gherkin
@smoke @layout
Feature: Complete homepage contract

  Background:
    When I add selectors from "homepage.json" file
    And I define css selectors:
      | cookie banner | #cookie-consent |

  Scenario Outline: <bp> viewport renders all sections correctly
    Given I am viewing the site on a <bp> screen
    And I am on homepage
    When I wait page is loaded

    # Visibility
    Then I see visible site header, primary nav, hero, cta, site footer
    And I don't see cookie banner

    # Layout
    And I see site header above hero
    And I see hero above site footer
    And I see primary nav inside of site header

    # Interactive
    When I click cta
    Then I should be on "/get-started"

    Examples:
      | bp  |
      | xs  |
      | md  |
      | xl  |
```

Five named selectors. Three breakpoints. Eight assertions. Zero raw CSS in the Gherkin. Hand this feature to a product owner and they can read every line. Hand it to a new QA engineer and they can extend it without learning your DOM first.

---

## The Philosophy

Named selectors turn tests into documentation. When you write `I see logo to the left of nav`, a product manager with no CSS knowledge understands the intent. When you write `.main-nav__container > ul > li:nth-child(3) a`, nobody does.

Webship-js ~2.0 adds this layer because long-running test suites always start to look like DOM archaeology. Registering selectors once, by name, restores readability without sacrificing precision.

And on teams that maintain many projects at once, names do one more job: they make suites comparable. The same `hero`, the same `primary nav`, the same layout contract -- twenty sites, one vocabulary. That is what turns a pile of test suites into a quality signal someone can actually read across a portfolio.

---

## Chapter Summary

- `webship-selectors.js` ships in webship-js ~2.0 and is loaded by default.
- Register named selectors inline (`When I add "..." selector for "..." css selector`), in bulk (`Given I define css selectors:`), or from JSON files (`When I add selectors from "file.json" file`). Files use the `{"css": {...}, "xpath": {...}}` shape and live in `tests/selectors/` by default.
- Spatial assertions: `above`, `below`, `left of`, `right of`, `inside of`, `outside of`, `over`, `not over`.
- Built-in breakpoints: `xs`, `sm`, `md`, `lg`, `xl`, `xxl`, `xxxl` (`xl` is the default). Override via `worldParameters.selectors.breakpoints` or `WEBSHIP_SELECTORS_BREAKPOINTS` -- remember the env JSON replaces the whole map. Scroll offset: 60px default, via `WEBSHIP_SELECTORS_OFFSET`.
- `I see <name> has focus`, `I move focus to "<field>" field`, and text-range selection steps drive keyboard-first flows.
- Click any named selector with `When I click <name>`.
- Layer your registry: shared file -> feature area -> scenario overrides. Last write wins.
- Multi-subject dispatch handles Cartesian assertions in one line.
- Debug the active registry with `I print css selectors` / `I print xpath selectors`.

---

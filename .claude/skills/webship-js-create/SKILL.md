---
name: webship-js-create
description: Author webship-js BDD `.feature` tests for a page or behaviour. Use when the user asks to "write tests for /<path>", "add scenarios for the login page", "test the contact form", "cover the checkout flow", "create a feature file", or names a route to test. Generates desktop + mobile scenarios with web-first assertions, registers named selectors, handles AJAX timing via BBR auto-settle, and writes numbered file names for CMS suites.
argument-hint: '[page-path]'
arguments: page
paths: tests/features/**/*.feature
allowed-tools: Bash(npx cucumber-js *) Bash(npm test *) Bash(npm run test*) Bash(curl *) Bash(ls *) Bash(cat *) Read Write Edit Glob Grep
---

# /webship-js-create — Author tests for a page

Create comprehensive BDD scenarios for a website page using
[webship-js](https://webship.co/docs/webship-js/2.0.x). Output:
`tests/features/<page>--<category>.feature` files with web-first
assertions, BBR smart waits, named selectors, and tag conventions
matching the suite.

`$page` — page path (`/contact`, `/login`, `/admin/content`) or feature
area name.

## Prerequisites

- Project scaffolded (`/webship-js-init` first if not).
- `cucumber.js` has `launchUrl` (or `LAUNCH_URL` env).

## Context loading (read before writing any file)

### 1. Operating principles

- `node_modules/webship-js/docs/12-ai-agent-guide.md` — REASONS canvas,
  Three Amigos, SPDD, golden rules.
- `node_modules/webship-js/docs/02-bbr-smart-waits.md` — wait-for-events
  philosophy, auto-settle, edge waits.
- `node_modules/webship-js/docs/03-selector-registry.md` — three ways to
  register selectors, CMS / framework presets, canonical names.
- `node_modules/webship-js/docs/14-recipes-cookbook.md` — 20 paste-and-go
  scenarios; closest one becomes the starting template.
- `node_modules/webship-js/docs/15-tag-conventions.md` — standard tags +
  CI lanes.
- `node_modules/webship-js/docs/16-ci-cd.md` — CI hooks.

If `node_modules/webship-js/docs/` is missing, fetch the same files from
https://github.com/webship/webship-js/tree/2.0.x.

### 2. Step source of truth (per-category)

webship-js is modular — one `<category>.steps.js` per category under
`node_modules/webship-js/tests/step-definitions/`. List what's installed
first, then read the files relevant to the page under test:

- `navigation`, `path`, `action` — go-to / back / pointer / drag / tap.
- `form`, `input`, `field` — fill / select / check / radio / multi-value /
  color / WYSIWYG / datetime.
- `wait` — BBR edge waits + bounded smart waits.
- `assertion`, `element`, `link` — visible-text + element existence + link
  href.
- `web-first` — auto-wait assertions (`"<sel>" should be visible/contain
  text/have value within N seconds`).
- `modal`, `dialog` — in-page modals + browser alert / confirm / prompt.
- `iframe` — switch / click / fill / switch back to root.
- `selectors` — registry + relative position + viewport breakpoints +
  focus.
- `responsive` — explicit viewport sizing.
- `keyboard` — key + chord presses.
- `screenshot`, `video` — capture.
- `cookie`, `storage` — cookies + local / session storage.
- `network` — mock URL responses, offline / online, request recording.
- `clock` — system time + advance.
- `auth` — basic auth + save / restore Playwright state.
- `api`, `rest`, `response` — HTTP requests + headers + JSON.
- `xml`, `yaml` — XML / YAML responses + JSON Schema match.
- `a11y` — axe-core audit + structural checks.
- `table` — row / column / sort / contents.
- `file-download` — verify downloaded files + zip contents.
- `metatag` — `<meta>` attribute assertions.
- `javascript` — JS-error capture.
- `debug` — `print current URL`, `print last response`.

Only use steps that exist in the loaded source.

### 3. Current project

- `cucumber.js` — `launchUrl`, `minWaitTime`, `users`, `selectors`.
- `playwright.config.ts` — browser + viewport.
- `tests/features/*.feature` — avoid duplicates; match existing naming.
- `tests/step-definitions/*.js` — existing custom steps.
- `tests/selectors/*.json` — preset files.

## Instructions

### Phase 1 — REASONS canvas

Before writing Gherkin, fill the canvas. Inline as `#` comments at top of
the `.feature` file:

```
# REASONS canvas
# R — Requirements: what behaviour proves done
# E — Entities:     domain nouns (Order, User, Cart)
# A — Approach:     Background + happy + sad scenarios
# S — Structure:    routes + selectors used
# O — Operations:   each scenario maps to one operation
# N — Norms:        a11y / i18n / perf / log expectations
# S — Safeguards:   security / rate-limit / failure modes
```

### Phase 2 — explore the page

1. Read `cucumber.js` for `launchUrl` (or use `LAUNCH_URL`).
2. Fetch the page HTML (`curl -sL <url>`) or use the Playwright MCP
   (`browser_navigate` + `browser_snapshot`). Identify:
   - Headings + content landmarks.
   - Forms (inputs, selects, checkboxes, radios, submit buttons).
   - Links (internal, external, `mailto:`, `tel:`).
   - Navigation, modals, AJAX-driven sections.
3. Reused named regions → register in `cucumber.js`
   `worldParameters.selectors.css` or via `Given I define css selectors:`
   inside the feature.
4. Check if a CMS / framework preset matches (Drupal, WordPress,
   Bootstrap, Tailwind, etc.) — load it instead of redefining canonical
   selectors.

### Phase 3 — choose file naming

Two conventions, pick whichever the existing suite uses:

**A — descriptive** (default for greenfield):

```
tests/features/login--page-load.feature
tests/features/login--form-empty-submit.feature
tests/features/login--form-invalid-password.feature
tests/features/login--form-valid-submission.feature
tests/features/login--links.feature
tests/features/login--a11y.feature
```

**B — numbered (CMS / multi-section suites)**:

```
tests/features/
├── 01-website-base-requirements/
├── 02-user-management/
│   ├── 02-01-request-new-password.feature
│   ├── 02-02-admins-can-create-users-and-assign-role-them.feature
│   ├── 02-03-user-login.feature
│   ├── 02-04-persistent-login.feature
│   ├── 02-05-user-protect.feature
│   └── 02-06-role-assign.feature
├── 03-admin-management/
├── 04-content-structure/
└── 05-content-management/
```

Two-digit section / two-digit feature. Predictable CI order; easy log
search. This pattern comes from large multi-role CMS suites.

### Phase 4 — tag for CI lanes

| Tag                                       | Meaning                                          |
|-------------------------------------------|--------------------------------------------------|
| `@critical` / `@smoke`                    | Must pass every PR.                              |
| `@auth`                                   | Touches authentication / sessions.               |
| `@security`                               | Asserts a Safeguard.                             |
| `@a11y`                                   | Accessibility (axe-core).                        |
| `@i18n`                                   | Multi-locale check.                              |
| `@perf`                                   | Performance budget.                              |
| `@flaky`                                  | Known unstable + open issue. Never long-term.    |
| `@wip`                                    | Work in progress; CI skips.                      |
| `@desktop` / `@mobile`                    | Viewport variants.                               |
| `@external`                               | Hits third-party; skip offline.                  |
| `@auth-setup`                             | One-shot; produces auth state JSON.              |
| `@video` / `@no-video`                    | Recording toggle when `video.mode = 'tag'`.      |
| `@js-fail` / `@js-warn` / `@js-off`       | Per-scenario JS-error mode.                      |
| `@local` `@development` `@staging` `@production` | Environment lanes.        |

### Phase 5 — write scenarios

Viewport via breakpoint registry (xs sm md lg xl xxl xxxl):

```gherkin
Given I am viewing the site on a "xl" screen     # desktop
Given I am viewing the site on a "xs" screen     # mobile
```

**Prefer web-first assertions** — they auto-wait, less flaky than
chaining explicit `wait` steps:

```gherkin
Then ".success-message" should be visible within 5 seconds
Then ".success-message" should contain text "Thanks for your message"
```

**Don't pre-pepper `And wait` everywhere.** BBR runs
`smartSettle(page, 1500)` after every action automatically. Only keep
waits for known animation durations, polling backends without fetch/XHR,
or `setInterval` heartbeats.

### Example — contact form validation (desktop + mobile)

```gherkin
Feature: Contact form validation
  As an anonymous user
  I want errors shown on invalid submissions
  So I know what to fix

  @desktop @validation @critical
  Scenario: Empty form shows errors on desktop
    Given I am viewing the site on a "xl" screen
    Given I am on "/contact"
    When I press "Submit"
    Then "#name-error" should be visible within 3 seconds
    And ".error" should contain text "Name field is required"

  @mobile @validation
  Scenario: Empty form shows errors on mobile
    Given I am viewing the site on a "xs" screen
    Given I am on "/contact"
    When I press "Submit"
    Then "#name-error" should be visible within 3 seconds
```

### Example — valid submission with data table

```gherkin
@desktop @submission
Scenario: Valid submission with all fields
  Given I am on "/contact"
  When I fill in the following:
    | Full name | Rajab Natshah       |
    | Email     | r@example.com       |
    | Subject   | Hello               |
    | Message   | Greetings from test |
  And I check "Accept terms"
  And I press "Submit"
  Then ".success" should be visible within 5 seconds
  And I should be on "/contact/thanks"
```

### Example — link attribute assertion

```gherkin
@desktop @links
Scenario: Email link uses mailto
  Given I am on "/contact"
  Then the "Email us" link should contain "mailto:hello@example.com"
```

### Example — a11y audit

```gherkin
@desktop @a11y
Scenario: Contact page meets WCAG AA
  Given I am on "/contact"
  Then the page should have a title
   And the page should declare a language
   And every form field should have an accessible label
   And the page should pass an accessibility audit at level "AA"
```

### Example — network-mocked SPA

```gherkin
@desktop @critical
Scenario: Dashboard renders mocked user list
  Given the URL "**/api/users" returns the JSON:
    """
    {"users": [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]}
    """
  When I am on "/users"
  Then ".user" should have a count of 2 within 5 seconds
   And I should see "Alice"
```

### Example — clock-mocked timeout

```gherkin
@desktop @security
Scenario: Session warning fires after 25 minutes
  Given the system time is "2026-01-01T00:00:00Z"
  Given I restore the auth state from "tests/auth/customer.json"
  Given I am on "/dashboard"
  When I advance the clock by 25 minutes
  Then ".session-warning" should be visible within 2 seconds
```

### Example — CMS content workflow

```gherkin
@javascript @local @development @staging @production
Feature: Content Publishing Workflow
  Scenario: Draft → Review → Publish
    Given I am a logged in user with the "Content editor" user
    When I go to "/admin/content/add"
     And I fill in "Title" with "New Product Launch"
     And I fill in "Body" with "We are launching..."
     And I press "Save as Draft"
    Then I should see "Status: Draft"
    When I press "Submit for Review"
    Then I should see "Status: Pending Review"
    Given I am a logged in user with the "Content admin" user
    When I go to "/admin/content"
    Then I should see "Pending Review" in the "New Product Launch" row
    When I click "Edit" in the "New Product Launch" row
     And I press "Publish"
    Then I should see "Status: Published"
    When I go to "/articles/new-product-launch"
    Then I should see "New Product Launch"
```

`Given I am a logged in user with the "<role>" user` reads
`worldParameters.users[<role>]` (set up in `/webship-js-init`). One
custom step replaces dozens of repeated login scenes — pattern from
multi-role CMS suites.

### Example — auth state restore (no repeat logins)

```gherkin
@desktop @critical
Scenario: Logout returns to homepage
  Given I restore the auth state from "tests/auth/admin.json"
   And I am on "/dashboard"
  When I click "Sign out"
  Then I should be on the homepage
   And I should see "Sign in"
```

Pattern: one `@auth-setup` scenario per role produces the JSON. Every
other scenario restores. No re-login overhead.

## Edge-case playbook

- **Prefer web-first.** When in doubt:
  `Then "<sel>" should be visible/contain text/have value within N seconds`.
- **AJAX / submits.** `When I press "Submit"` triggers BBR auto-settle
  (1500 ms). For longer flows chain `And I wait for AJAX to finish` or a
  web-first wait.
- **Auto-dismissing messages.** Lower `worldParameters.minWaitTime.page`
  (`3000 → 500`) so assertions fire before fade.
- **Rate limiting / flood control.** Write a custom step polling for
  either outcome (success OR rate-limit text).
- **HTML attributes.** `the response should contain` checks **text**, not
  attributes. Use link-by-attribute
  (`the "contact" link should contain "mailto:" by its "href" attribute`)
  or element-attribute
  (`the element ".cta" with the attribute "data-test" and the value "primary" should exist`).
- **Duplicate text.** Scope with named selectors or web-first:
  `Then "#success" should contain text "Saved" within 3 seconds`.
- **Iframes.** `When I switch to the iframe "<sel>"` first; run
  `... inside the iframe` steps; then `When I switch to the root
  document`. Skip the root-switch and follow-up steps hang.
- **Time-dependent UI.** `Given the system time is "<iso>"` (ISO 8601 —
  bad format throws). `When I advance the clock by N minutes`.
- **External APIs in CI.** Mock with
  `Given the URL "<pattern>" returns the JSON:` instead of hitting real
  services. `**` glob matches any host/path.
- **Keyboard.** Stick to Playwright key identifiers (`Enter`, `Tab`,
  `Control+S`). Bad names throw `Unknown key: <X>`.
- **File downloads.** Smoke a known-good fixture first or mock the route
  (`Given the URL "..." returns status 200 with body "..."`); 404 throws
  `Download failed with status 404`.
- **Video.** `When I start video recording` before save; or use
  `WEBSHIP_VIDEO=on-failure` and let the runner handle it.

## Phase 6 — run

```bash
# all tests
npm test
# single file
npx cucumber-js --config cucumber.js tests/features/$page--form-empty-submit.feature
# tag filter
npx cucumber-js --config cucumber.js --tags "@desktop and @validation"
# different browser
BROWSER=firefox npm test
# watch the browser (debug)
npm run test:headed
# fastest run (no slow-mo)
npm run test:fast
# record video of failures
WEBSHIP_VIDEO=on-failure npm test
# CI smoke gate
npx cucumber-js --tags "@critical and not @wip"
```

DDEV:

```bash
ddev npm run test:chromium
ddev exec npx cucumber-js --config cucumber.js tests/features/$page--form-empty-submit.feature
```

## Phase 7 — report

Auto-generated after each run. Regenerate manually:

```bash
npm run generate-reports
# HTML + PDF
npx generate-reports --format all
```

Report paths:

- `tests/reports/cucumber_report.json`
- `tests/reports/cucumber_report.html`
- `tests/reports/cucumber_report.pdf` (when requested)

Summarize: scenarios total, pass/fail counts, failure root causes + fixes.

## Custom step skeleton

```js
// tests/step-definitions/custom.js
const { Given, When, Then } = require('@cucumber/cucumber');

When(/^(I |we )*wait for rate limit or success$/, async function () {
  await this.page.waitForFunction(() => {
    const body = document.body.innerText;
    return /Thanks|too many requests/i.test(body);
  }, { timeout: 30000 });
});
```

World object: `this.page`, `this.context`, `this.playwrightBrowser`,
`this.launchUrl`, `this.minWaitTime`, `this.assetsFolder`,
`this.parameters` (full `worldParameters`).

Multi-role auth helper pattern:

```js
Given(/^I am a logged in user with( the)*( username)* "([^"]*)?"( user)*$/, async function (_, __, username) {
  const users = this.parameters.users;
  if (!(username in users)) throw new Error(`${username} not in worldParameters.users`);
  const loginName = users[username].username || username;
  await this.page.goto(this.launchUrl + '/user/login', { waitUntil: 'domcontentloaded' });
  await this.page.fill('#edit-name', loginName);
  await this.page.fill('#edit-pass', users[username].password);
  await this.page.click('#edit-submit');
  await this.page.waitForLoadState('domcontentloaded');
});
```

On multi-role CMS suites that step
collapses dozens of repeated login scenes into one helper plus a JSON
registry.

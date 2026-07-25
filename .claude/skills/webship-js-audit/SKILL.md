---
name: webship-js-audit
description: Audit webship-js `.feature` files and custom step definitions for patterns and anti-patterns. Use when the user asks to "review my feature file", "audit my tests", "lint my Gherkin", "check this scenario", "is this step clean", "is this an anti-pattern", "find smells in tests/", "review custom.js". Flags sleep-driven waits, god scenarios, brittle selectors, implementation testing, premature custom steps, leaked module state, and other documented smells; suggests the fix.
argument-hint: '[path-or-glob]'
arguments: target
paths: tests/features/**/*.feature,tests/step-definitions/**/*.js
allowed-tools: Bash(ls *) Bash(cat *) Bash(grep *) Bash(rg *) Bash(wc *) Read Glob Grep
---

# /webship-js-audit — Patterns + anti-patterns review

Read `.feature` files and custom step definitions and report violations
against the documented webship-js patterns + anti-patterns. Output is
file:line — severity — pattern — fix suggestion.

`$target` — file or glob (`tests/features/login.feature`,
`tests/features/**/*.feature`, `tests/step-definitions/custom.js`,
`tests/`). Defaults to `tests/features/**/*.feature
tests/step-definitions/**/*.js`.

## Context loading (source of truth)

1. **Installed step catalog** — to verify "is there already a built-in"
   for any candidate custom step:

   ```bash
   ls node_modules/webship-js/tests/step-definitions/*.steps.js
   grep -rEh '^(Given|When|Then)\(/?\^?' node_modules/webship-js/tests/step-definitions/
   ```

2. **Patterns + philosophy docs** at
   `node_modules/webship-js/docs/`:
   - `12-ai-agent-guide.md` — Anti-Patterns Hall of Fame, ten Golden
     Rules, Recipe AI-2 (step gen), AI-5 (when tests go bad).
   - `02-bbr-smart-waits.md` — wait-for-events; no sleep-then-check.
   - `03-selector-registry.md` — named selectors + CMS / framework
     presets + canonical names.
   - `05-web-first-assertions.md` — prefer auto-wait matchers.
   - `06-network-and-dialogs.md` — `**/api/...` globs + dialog handler
     ordering.
   - `07-auth-state.md` — restore vs re-login.
   - `08-clock-mocking.md` — ISO 8601 input.
   - `10-accessibility.md` — axe excludes + structural checks.
   - `11-debugging.md` — diagnoses table.
   - `15-tag-conventions.md` — tag hygiene rules.

3. **Project state**:
   - `cucumber.js` — `worldParameters` keys actually configured.
   - `tests/features/**/*.feature` — content to audit.
   - `tests/step-definitions/**/*.js` — custom steps to audit.
   - `tests/selectors/*.json` — preset files referenced.

## What this skill audits

### A. Feature files (`*.feature`)

#### Patterns to confirm present

| Pattern                                  | Source of rule                       |
|------------------------------------------|--------------------------------------|
| One scenario = one behaviour (DAMP).     | `12-ai-agent-guide.md` Golden Rules. |
| Each scenario creates its own data.      | same.                                |
| `Background` is short (≤ 5 lines).       | `13-faq.md`.                         |
| Web-first assertion where a wait+check pair appears (`Then "<sel>" should be visible within N seconds`). | `05-web-first-assertions.md`. |
| Named selector (or canonical name) when text is ambiguous or reused. | `03-selector-registry.md`. |
| Role-based locator (`click the "Sign in" button`) when the role is obvious. | `05-web-first-assertions.md`. |
| Restore auth state at scenario / Background start; no inline re-login. | `07-auth-state.md`. |
| Mock external APIs with `Given the URL "<glob>" returns ...` in CI scenarios. | `06-network-and-dialogs.md`. |
| Dialog handler registered **before** the action that triggers it.       | `06-network-and-dialogs.md`. |
| Iframe scope opened then closed (`switch to the iframe ...` then `switch to the root document`). | `iframe.steps.js`. |
| Clock dates ISO 8601.                                                   | `08-clock-mocking.md`. |
| Video recording started before `save the current video as ...`.          | `video.steps.js`. |
| Tag every scenario with at least one of `@critical / @smoke / @auth / @security / @a11y / @i18n / @perf / @desktop / @mobile / @flaky / @wip`. | `15-tag-conventions.md`. |
| `@flaky` carries an open issue link.                                    | `15-tag-conventions.md`. |
| No `@skip` tag — use `@wip` instead.                                    | `15-tag-conventions.md`. |

#### Anti-patterns to flag

| Anti-pattern                                                                    | Severity | Fix                                                                                              |
|---------------------------------------------------------------------------------|----------|--------------------------------------------------------------------------------------------------|
| `When I wait N seconds` immediately followed by `Then I should ...`.            | WARN     | Replace with web-first: `Then "<sel>" should be visible within N seconds`.                       |
| Pre-peppered `And I wait …` before every action.                                | WARN     | Drop. BBR auto-settle covers this. Keep only for known-duration animations or polling backends.  |
| `wait` after `I press` when the next step is a web-first matcher.               | INFO     | Drop the wait. Web-first auto-retries until match or budget.                                     |
| Multiple distinct behaviours in one Scenario.                                   | WARN     | Split into separate scenarios.                                                                   |
| Scenario depends on another scenario's side effect.                             | CRITICAL | Make scenario self-contained; reset state in Background.                                         |
| `Then the database should …` / `Then the cache should …`.                       | CRITICAL | Test what the user experiences — UI or API.                                                      |
| `Then the response should contain "mailto:..."` (text-only step on HTML attr).  | CRITICAL | Use link-by-attribute or element-attribute form.                                                 |
| Hard-coded brittle CSS in feature text (`Then ".css-9a3b2x"`).                  | WARN     | Register a named selector; reference by name.                                                    |
| `Background:` longer than 5 lines.                                              | INFO     | Extract setup into a custom step or auth-state restore.                                          |
| Conditional logic inside Gherkin (`Then if discount > 5 ...`).                  | CRITICAL | Push the branch into the step definition; use clean Gherkin sentences.                           |
| `@flaky` without `@critical` workaround or referenced open issue.                | WARN     | Each `@flaky` needs an issue; never long-term.                                                   |
| `@skip` tag present.                                                            | WARN     | Replace with `@wip` and run `--tags "not @wip"`.                                                 |
| Custom-named tags duplicating standards (`@p0`, `@blocker`, `@must-pass`).      | INFO     | Use `@critical`.                                                                                 |
| Dialog handler registered **after** the trigger action.                         | CRITICAL | Move `Given I will accept the next dialog` before `When I click "Delete"`.                       |
| Iframe-scoped step without prior `switch to the iframe`.                        | CRITICAL | Add the switch; remember to close with `switch to the root document`.                            |
| `Given the system time is "not-a-date"` (non-ISO 8601).                         | CRITICAL | Use ISO 8601 (`"2026-01-01T00:00:00Z"`).                                                         |
| `When I save the current video as ...` without prior `start video recording`.   | CRITICAL | Add `start video recording` or switch to `WEBSHIP_VIDEO=on-failure`.                             |
| Bad Playwright key identifier in `press the key "<X>"`.                         | CRITICAL | Use Playwright names (`Enter`, `Tab`, `Control+S`).                                              |
| Login form filled per scenario instead of `restore the auth state`.             | WARN     | Save state once via `@auth-setup`; restore everywhere else.                                      |
| External API call from a CI scenario without `Given the URL "<pattern>" returns ...`. | WARN | Mock with a `**/path` glob; ship offline-safe scenarios.                                         |
| Mobile or desktop scenario without `Given I am viewing the site on a "<bp>" screen`. | INFO | Pin a breakpoint; CI defaults differ.                                                            |
| Scenario name describes implementation (`Scenario: GET /api/users returns 200`). | INFO   | Rename to user-visible behaviour.                                                                |

### B. Custom step files (`tests/step-definitions/*.js`)

#### Patterns to confirm present

| Pattern                                                                   | Source of rule                  |
|---------------------------------------------------------------------------|---------------------------------|
| Regex uses `(I |we )*` (not single-pronoun Cucumber Expressions).         | `12-ai-agent-guide.md` AI-2.    |
| JSDoc block above each step with ≥ 5 `Example #N:` Gherkin lines.         | `12-ai-agent-guide.md` AI-2.    |
| Every example actually matches the step regex.                            | `12-ai-agent-guide.md` AI-2.    |
| State lives on `this` (per-scenario World).                               | core webship-js convention.     |
| Custom action steps call `this.smartSettle()` (where helper exists).      | `02-bbr-smart-waits.md`.        |
| Fixtures (lists of expected texts, selector groups) live in a `tests/fixtures/*.js` module, not on `this`. | composition principle.          |
| One step file per domain concern (`chrome.steps.js`, `cart.steps.js`, ...) once `custom.js` exceeds ~10 steps. | maintainability rule.           |

#### Anti-patterns to flag

| Anti-pattern                                                                                    | Severity | Fix                                                                                                                                            |
|-------------------------------------------------------------------------------------------------|----------|------------------------------------------------------------------------------------------------------------------------------------------------|
| **Re-implementing a built-in.** Custom step does one Playwright call (`page.click(...)`) for a built-in case (`I press "Submit"`). | CRITICAL | Delete the custom step; use the built-in.                                                                                                      |
| **Premature extraction.** Custom step used only once.                                            | WARN     | Inline the steps in the feature file. Three-use rule: extract only after the third call site.                                                   |
| **Over-specific regex.** Step regex contains domain literals (`"Webship-js" article on blog`). | WARN     | Generalise with capture groups (`the article "([^"]+)" should be visible on the blog`).                                                         |
| **Module-level state leak.** `let lastOrderId;` outside the `Given/When/Then` callbacks.        | CRITICAL | Move to `this.lastOrderId` so each scenario gets a fresh World.                                                                                 |
| **Business logic hidden inside a single mega-step.** 80-line `When a valid order is placed`.    | CRITICAL | Decompose into composable steps using existing built-ins; let the feature file read like a spec.                                                |
| **Bad function arity.** Regex captures don't match callback parameters (`function has 0 arguments, should have N`). | CRITICAL | Add the captured-group params to the callback signature in order.                                                                               |
| **Inline conditional logic** inside a step body (`if (mode === 'foo') ... else ...`).            | WARN     | Split into two steps named after the user-visible behaviour each branch produces.                                                               |
| **Implementation-flavoured naming** (`header_assertion_should_pass`, `test_check_chrome`).      | INFO     | Rename to a domain sentence (`I should have a working header`).                                                                                 |
| **Custom step for a negative** when the built-in negation works.                                 | INFO     | Use `Then I should not see "X" in the "header" region` instead of a `working-broken-header` step.                                              |
| **Selector + fixture text mixed in one file**.                                                  | INFO     | Selectors → step file. Lists of expected texts → `tests/fixtures/*.js` module.                                                                  |
| **Step missing `this.smartSettle()` after an action** when one is available.                    | INFO     | Reuse the helper so custom action steps wait the same way built-ins do.                                                                         |
| **Dead custom step** — not referenced by any `.feature` file.                                    | INFO     | Delete it. Stale custom vocabulary confuses newcomers.                                                                                          |

### C. Cross-file checks

| Check                                                                              | Severity | Fix                                                                                                                                 |
|------------------------------------------------------------------------------------|----------|-------------------------------------------------------------------------------------------------------------------------------------|
| Feature file references a custom step that no `.steps.js` defines.                 | CRITICAL | Add the step or fix the phrasing.                                                                                                   |
| Feature file references a named selector not present in `worldParameters.selectors.css` / `.xpath` / loaded preset JSON. | CRITICAL | Register the selector or load the preset.                                                                                            |
| Auth restore points at a missing `tests/auth/<role>.json` file.                    | CRITICAL | Run the matching `@auth-setup` scenario to generate the file.                                                                       |
| `worldParameters.users["<role>"]` referenced from `Given I am a logged in user with the "<role>" user` but missing from `cucumber.js`. | CRITICAL | Add the entry; otherwise the custom step throws.                                                                                    |
| `WEBSHIP_VIDEO=on-failure` documented but `worldParameters.video.mode` set to `'off'` in `cucumber.js` (or block missing).            | INFO     | Align config and env so the recording actually runs.                                                                                |

## Decision flowchart (when reviewing a custom step)

```
Is the behaviour already covered by a built-in step?
├── Yes  → Flag CRITICAL "re-implementing a built-in".
└── No   → Continue.

Is the same multi-step block in 3+ features?
├── Yes  → Custom step justified.
└── No   → Flag WARN "premature extraction".

Does the step name describe a domain concept the built-ins can't?
├── Yes  → Custom step justified.
└── No   → Flag INFO "rename to domain sentence".

Will the implementation centralise a fragile selector or invariant?
├── Yes  → Custom step justified.
└── No   → Inline the steps; flag WARN if extraction premature.
```

Three-line test for extraction: **(a)** 3+ uses, **(b)** 3+ inline
lines, or **(c)** 3+ engineers ask "what does that mean?" — extract.
None tripped — inline is fine.

## How to run

### Per-file

```
/webship-js-audit tests/features/login.feature
/webship-js-audit tests/step-definitions/custom.js
```

### Whole tree (default)

```
/webship-js-audit
```

### Subset

```
/webship-js-audit tests/features/checkout/**/*.feature
```

## Output format

```
## Audit summary

Scanned:  M features, N step files.
Findings: A CRITICAL, B WARN, C INFO.

## Findings

### tests/features/login--form-validation.feature
- L12  CRITICAL — Sleep-then-check
       `And I wait 3 seconds` then `Then I should see "Error"`.
       Fix: `Then ".error" should be visible within 3 seconds`.

- L22  WARN     — Brittle CSS in feature text
       `Then ".css-9a3b2x" should be visible`.
       Fix: register a named selector in cucumber.js, e.g. `error notice`,
       then `Then "error notice" should be visible within 3 seconds`.

### tests/step-definitions/custom.js
- L17  CRITICAL — Re-implementing a built-in
       `When I click the submit button` wraps a single page.click.
       Fix: delete; use built-in `When I press "Submit"`.

- L42  CRITICAL — Module-level state leak
       `let lastOrderId;` outside callbacks.
       Fix: move to `this.lastOrderId` so the World scopes per scenario.

### Cross-file
- Auth state `tests/auth/admin.json` referenced by 4 features but file missing.
  Fix: run `npx cucumber-js --tags @auth-setup` to regenerate.

## Top three to fix first

1. tests/step-definitions/custom.js:17 — delete duplicate built-in (1 min).
2. tests/features/login--form-validation.feature:12 — convert to web-first (2 min).
3. tests/features/login--form-validation.feature:22 — register named selector (5 min).
```

## When the audit finds nothing

State that clearly. Print "No findings — suite follows webship-js
patterns." Don't manufacture nits.

## Walk-back / maintenance

Custom steps decay. Audit quarterly. For each custom step ask:

1. Is the step still in use? (grep `tests/features/` for the phrasing.)
2. Does its implementation match the current product?
3. Has its name drifted from the team's domain language?

If any answer is "no" — fix or delete. A wrong custom step is worse
than no custom step; it produces false confidence.

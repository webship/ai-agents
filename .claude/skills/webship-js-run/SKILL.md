---
name: webship-js-run
description: Run the webship-js suite, generate the HTML / PDF report, and analyze failures. Use when the user asks to "run tests", "run the suite", "execute @critical", "generate report", "show me failures", "what failed", or names a tag / feature path to run. Handles tag filtering, file targeting, headed-mode debug, video recording, JS-error capture, and produces a root-cause summary for every failure.
argument-hint: '[tags-or-feature-path]'
arguments: filter
allowed-tools: Bash(npm test*) Bash(npm run test*) Bash(npx cucumber-js*) Bash(npx generate-reports*) Bash(npx playwright *) Bash(ddev *) Bash(node *) Bash(ls *) Bash(cat *) Read Glob Grep
---

# /webship-js-run — Run tests, generate report, analyze failures

Run the [webship-js](https://webship.co/docs/webship-js/2.0.x) suite,
produce HTML / PDF reports, and summarize results with root-cause
analysis of every failure.

`$filter` — optional tag expression (`@critical`, `"@desktop and @a11y"`,
`"@critical and not @flaky"`) or a feature path
(`tests/features/login.feature`).

## Context loading

### 1. Docs

Read the local installed copy at
`node_modules/webship-js/docs/` (or the source at
https://github.com/webship/webship-js/tree/2.0.x):

- `11-debugging.md` — auto-screenshot, headed mode, JS-error capture,
  HTML / PDF report flags, scenario timing, trace viewer, common
  diagnoses table.
- `15-tag-conventions.md` — CI lane phrasings (`@critical`, `@a11y`,
  `@flaky`, `@auth`, `@wip`).
- `16-ci-cd.md` — provider-specific recipes (GitHub Actions, GitLab,
  Bitbucket, CircleCI, Jenkins, Azure, AWS CodeBuild, Google Cloud
  Build, TeamCity, Drone, Semaphore, Harness, Bamboo, Codefresh,
  Octopus, Travis).
- `02-bbr-smart-waits.md` — why explicit `wait Ns` shouldn't be the fix.
- `05-web-first-assertions.md` — preferred replacement for sleep-then-check.

### 2. Step source per category

Recent webship-js is modular — one `<category>.steps.js` per category
under `node_modules/webship-js/tests/step-definitions/`. Read whichever
file owns the failing step:

```bash
ls node_modules/webship-js/tests/step-definitions/
# a11y action api assertion auth clock cookie debug dialog element field
# file-download form iframe input javascript keyboard link metatag modal
# navigation network path response responsive rest screenshot scroll
# selectors storage table video wait web-first xml yaml
```

Plus `node_modules/webship-js/bin/generate-reports.js` — report
generator.

### 3. Current project

- `cucumber.js` — `launchUrl`, `minWaitTime`, `users`, `selectors`,
  `screenshot`, `video`, `javascript`, `diffy`.
- `playwright.config.ts` — browser, viewport, slow-mo.
- `tests/features/*.feature` — features under test.
- `tests/step-definitions/*.js` — custom steps.
- Existing `tests/reports/cucumber_report.json` (if previously run).

## Instructions

### 1. Verify install

```bash
node -v                                          # >= 20
ls node_modules/webship-js                       # installed
cat cucumber.js                                  # launchUrl
node -e "console.log(require('webship-js/package.json').version)"
```

DDEV: prefix each with `ddev exec`.

### 2. Run the suite

Plain Node.js:

```bash
# all tests
npm test
# tag filter (from $filter)
npx cucumber-js --config cucumber.js --tags "$filter"
# single file
npx cucumber-js --config cucumber.js tests/features/<file>.feature
# named scenario
npx cucumber-js --config cucumber.js --name "Successful login"
# per browser
npm run test:chromium
npm run test:firefox
npm run test:webkit
# watch the browser (HEADLESS=false + slow-mo)
npm run test:headed
# fastest run (SLOW_MO=0)
npm run test:fast
# combine env
HEADLESS=false SLOW_MO=500 BROWSER=firefox npm test
# colored CI logs (cucumber-js v10+)
FORCE_COLOR=1 npm test
# parallel + retries (CI)
SLOW_MO=0 npx cucumber-js --parallel 4 --retry 1 --retry-tag-filter @flaky
# CI smoke gate
npx cucumber-js --tags "@critical and not @wip"
# Full suite (no WIP, no auth-setup)
npx cucumber-js --tags "not @wip and not @auth-setup"
# A11y lane
npx cucumber-js --tags "@a11y"
# Flaky lane (separate report, allowed to fail)
npx cucumber-js --tags "@flaky" --retry 2
# Pre-release security gate
npx cucumber-js --tags "@security or @auth"

# video recording
WEBSHIP_VIDEO=on npm test                       # every scenario
WEBSHIP_VIDEO=on-failure npm test               # only failures
WEBSHIP_VIDEO=tag npm test                      # only @video-tagged

# JS-error reporter
WEBSHIP_JS_ERROR_MODE=fail npm test             # fail any scenario with JS errors
WEBSHIP_JS_ERROR_LEVELS=error,warning npm test  # capture warnings too
WEBSHIP_JS_ERROR_IGNORE='third-party-sdk' npm test

# disable auto-settle (rarely needed)
WEBSHIP_AUTO_SETTLE=off npm test

# disable auto HTML report
WEBSHIP_REPORT_DISABLE=1 npm test
```

DDEV:

```bash
ddev npm run test:chromium
ddev exec npx cucumber-js --config cucumber.js --tags "$filter"
ddev exec npx cucumber-js tests/features/<file>.feature
```

### 3. Report

HTML auto-generates after every run unless `WEBSHIP_REPORT_DISABLE=1`.
Regenerate from the JSON:

```bash
npm run generate-reports
# or
npx generate-reports
# HTML + PDF in one shot
npx generate-reports --format all
# PDF only — Letter, landscape, slim margin
npx generate-reports --format pdf --pdf-format Letter --pdf-landscape --pdf-margin 10mm
# Branded PDF with header / footer
npx generate-reports --format pdf \
  --pdf-header '<div style="font-size:10px;width:100%;text-align:center;">Acme Q3 Regression</div>' \
  --pdf-footer '<div style="font-size:10px;width:100%;text-align:center;"><span class="pageNumber"></span>/<span class="totalPages"></span></div>'
```

Flags (per docs `11-debugging.md`):

| Flag                         | Purpose                                                           |
|------------------------------|-------------------------------------------------------------------|
| `--format html\|pdf\|all`    | Repeatable. Default `html`.                                       |
| `--pdf-format`               | `Letter`, `Legal`, `A3`, `A4`, `A5` (default `A4`).               |
| `--pdf-landscape`            | Landscape orientation.                                            |
| `--pdf-margin`               | CSS margin for all sides (default `20mm`).                        |
| `--pdf-header` / `--pdf-footer` | HTML templates (use `<span class="pageNumber">`).             |
| `--pdf-no-background`        | Disable print backgrounds.                                        |
| `--pdf-scale`                | Scale factor 0.1–2.0.                                             |
| `--pdf-out`                  | Override PDF output path.                                         |

Paths:

- `tests/reports/cucumber_report.json` — raw Cucumber output.
- `tests/reports/cucumber_report.html` — HTML dashboard.
- `tests/reports/cucumber_report.pdf` — PDF (when requested).
- `screenshots/` — failure screenshots (prefix `failed_`).
- `videos/` — per-scenario recordings when `WEBSHIP_VIDEO != off`.

### 4. Summarize

Read `tests/reports/cucumber_report.json` and produce:

```
## Test results

| Metric    | Count |
|-----------|-------|
| Features  | X     |
| Scenarios | X     |
| Passed    | X     |
| Failed    | X     |
| Skipped   | X     |
| Steps     | X     |
| Duration  | Xs    |

### Failures
- **Feature / Scenario** — "name"
  - **Step**: `When I press "Submit"`
  - **Error**: `the page took too long to respond` (verbatim)
  - **Screenshot**: `screenshots/failed_<...>.png`
  - **Video**: `videos/<...>.webm` (if recording was on)
  - **Root cause**: AJAX race — click fired but assertion ran before
    DOM update settled
  - **Fix**: replace `Then I should see "Saved"` with web-first
    `Then ".saved" should be visible within 3 seconds`

### Reports
- HTML: `tests/reports/cucumber_report.html`
- JSON: `tests/reports/cucumber_report.json`
- PDF:  `tests/reports/cucumber_report.pdf` (if requested)
```

### 5. Root-cause checklist

Map every failure to one of these patterns first (matches the
error-smoke log):

| Error / symptom                                        | Root cause                                                  | Fix                                                                                                                                |
|--------------------------------------------------------|-------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| `the page took too long to respond`                    | Selector / iframe target missing; 30s Playwright timeout fires | Convert to web-first `Then "<sel>" should be visible within N seconds`. For iframes, check the locator + remember to switch to the root document before subsequent steps. |
| `Unknown key: <X>`                                     | Bad keyboard key identifier                                 | Use Playwright key names (`Enter`, `Tab`, `Control+S`). See `keyboard.steps.js`.                                                   |
| `Download failed with status 404`                      | Download URL missing                                        | Smoke a known-good fixture first; mock the route with `Given the URL "..." returns status 200 with body "..."`.                    |
| `Could not set the system time to '<X>'`               | Clock step needs ISO 8601                                   | Validate date format before `Given the system time is "<iso>"`.                                                                    |
| `Error: function has N arguments, should have M`       | Step regex captures don't match callback arity              | Add the captured-group params to the callback signature in the custom step.                                                        |
| `recording is not active`                              | Video save without prior start                              | `When I start video recording` before save. Or `WEBSHIP_VIDEO=on-failure` to skip the manual steps.                                |
| Timeout on text / element                              | Missing AJAX wait after `I press` / `I click` / submit      | Use web-first; or `And I wait for AJAX to finish` for legacy steps.                                                                |
| Wrong element matched                                  | Ambiguous text                                              | Register a named selector; use positional form (`When I click login button`).                                                      |
| Text seen by curl but not by test                      | Auto-dismissed message                                      | Lower `worldParameters.minWaitTime.page` (3000 → 500).                                                                             |
| Selector lookup failed                                 | JSON preset not loaded                                      | Confirm `worldParameters.selectors.files` lists the JSON file.                                                                     |
| Iframe content invisible                               | Test never switched into the iframe                         | `When I switch to the iframe "<sel>"`; remember to switch back to root.                                                            |
| HTTP 403 / 429                                         | Rate limit / flood control                                  | Custom step accepting either outcome (success OR rate-limit text).                                                                 |
| a11y false positive                                    | Third-party embed flagged by axe-core                       | `Then the page should pass an accessibility audit excluding ".third-party"`.                                                       |
| JS-error noise                                         | Vendor SDK throws                                           | `WEBSHIP_JS_ERROR_IGNORE='vendor-sdk'` or `@js-warn` / `@js-off` tag.                                                              |
| Playwright browser missing                             | First-time install skipped                                  | `npx playwright install --with-deps chromium` (or `ddev exec` same).                                                               |
| `function timed out`                                   | Cucumber's outer timeout fires before Playwright's          | Bump `timeout` in `cucumber.js` (default 45000).                                                                                   |
| Passes locally, fails in CI                            | Different viewport, browser, or speed                       | Pin viewport (`Given I set the viewport to the "desktop" breakpoint`); reproduce locally with `BROWSER=<ci-browser>`; bump web-first budget to `within 15 seconds`. |
| Auth state expired                                     | Saved cookies past lifetime                                 | Re-run `--tags @auth-setup` to regenerate `tests/auth/<role>.json`.                                                                |

### 6. Per-scenario timing

For per-scenario breakdown (slow tests), parse the JSON report:

```bash
node -e "
const r = require('./tests/reports/cucumber_report.json');
const s = [];
r.forEach(f => f.elements.forEach(sc => {
  const ms = sc.steps.reduce((a,b)=>a+(b.result?.duration||0),0)/1e6;
  s.push({n: sc.name, ms});
}));
s.sort((a,b)=>b.ms-a.ms).slice(0,10).forEach(x =>
  console.log(x.ms.toFixed(0).padStart(6), 'ms', x.n));
"
```

Cross-reference each failing step against its regex in the matching
`<category>.steps.js` to confirm the phrasing is valid on the installed
version. If `friendly()` / `humanize()` error filtering is present in
the installed package, the test output already explains the root cause
— surface it verbatim before adding analysis.

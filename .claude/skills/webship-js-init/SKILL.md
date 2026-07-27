---
name: webship-js-init
description: Scaffold a webship-js test project for a target site. Use when the user asks to set up webship-js, initialize tests, add Playwright + Cucumber-js BDD testing, or install the ddev-webship-js add-on. Triggers on phrases like "set up webship-js", "init webship", "scaffold tests", "add BDD tests for <url>", "set up testing for this DDEV project". Idempotent — preserves existing files.
argument-hint: '[URL | --ddev]'
arguments: target
allowed-tools: Bash(npm *) Bash(npx *) Bash(node *) Bash(ddev *) Bash(ls *) Bash(cat *) Read Write Edit Glob Grep
---

# /webship-js-init — Initialize a webship-js test project

Scaffold a [webship-js](https://webship.co/docs/webship-js/2.0.x) project
(Playwright + Cucumber-js) for any website. Idempotent — keeps user files
unless `--force` is asked. Works for plain Node.js or DDEV.

`$target` — target URL (`https://example.com`) or `--ddev` for the
[`ddev-webship-js`](https://github.com/webship/ddev-webship-js) add-on.

## Context loading (read before writing any file)

### 1. Official docs

Fetch:

- https://webship.co/docs/webship-js/2.0.x
- https://webship.co/docs/webship-js/2.0.x/install-webship-js
- https://webship.co/docs/webship-js/2.0.x/global-settings
- https://webship.co/docs/webship-js/2.0.x/commands

### 2. Installed package (if present)

If `node_modules/webship-js/` exists, read the source of truth:

- `node_modules/webship-js/bin/init-webship.js` — what `npx init-webship-js`
  actually writes.
- `node_modules/webship-js/cucumber.js` — default `worldParameters`.
- `node_modules/webship-js/playwright.config.ts` — default browser config.
- `node_modules/webship-js/package.json` — scripts + deps.
- `node_modules/webship-js/docs/00-quick-start.md` and
  `node_modules/webship-js/docs/01-getting-started.md`.

If missing, fetch the same files from
https://github.com/webship/webship-js/tree/2.0.x.

### 3. Existing project (re-runs)

- `cucumber.js` — current config (don't overwrite without consent).
- `playwright.config.ts` — current browser config.
- `tests/features/*.feature` — user-authored features.
- `tests/step-definitions/*.js` — user custom steps.
- `package.json` — merge, never clobber.

## Instructions

### Path A — plain Node.js (`$target` is a URL)

1. Create project dir if none exists. Default name from the host
   (`test-example-com`).
2. Install + scaffold:

   ```bash
   cd <project-dir>
   npm install --no-save webship-js
   npx init-webship-js                       # idempotent
   # npx init-webship-js --force             # only when user asked
   # npx init-webship-js --skip-browsers     # skip Playwright browser install
   ```

3. Point at the target site. Prefer env var:

   ```bash
   LAUNCH_URL=$target npm run test:chromium
   ```

   Or edit `cucumber.js`:

   ```js
   worldParameters: {
     launchUrl: process.env.LAUNCH_URL || '$target',
     // ...
   }
   ```

4. Smoke-test:

   ```bash
   LAUNCH_URL=$target npm run test:chromium
   ```

### Path B — DDEV project (`$target` is `--ddev`)

1. From the DDEV project root:

   ```bash
   ddev add-on get webship/ddev-webship-js
   ddev restart
   ```

   The add-on: adds Playwright + browsers to the web container, scaffolds
   on post-start (preserves existing files), sets
   `LAUNCH_URL=${DDEV_PRIMARY_URL}`, and replaces the default sample with
   a DDEV-universal one.

2. Run tests inside the container:

   ```bash
   ddev npm run test:chromium
   ddev npm run test:firefox
   ddev npm run test:webkit
   ```

3. Re-scaffold inside an existing DDEV container:

   ```bash
   ddev exec npx init-webship-js
   ddev exec npx init-webship-js --force
   ```

## What gets scaffolded

```
<project>/
├── cucumber.js                 # worldParameters: launchUrl, selectors, screenshot, video, javascript, diffy
├── playwright.config.ts        # browser + viewport (null + --start-maximized for chromium)
├── tsconfig.json               # tsx loader
├── package.json                # scripts: test, test:chromium/firefox/webkit, test:headed, test:fast, generate-reports
├── screenshots/                # auto on failure
├── videos/                     # when worldParameters.video.mode != 'off'
└── tests/
    ├── features/check-homepage.feature + README.md
    ├── step-definitions/custom.js + README.md
    ├── selectors/              # JSON selector files (project-wide presets)
    └── reports/README.md
```

## Scaffold conventions to verify (don't assume)

Things to look for after install — don't recommend what isn't installed:

- `requireModule: ['tsx/cjs']` (replaces older `ts-node/register`).
- Step `timeout` raised above Playwright's default so Playwright errors
  surface first.
- Cucumber-js v10+ — color via `FORCE_COLOR` env (`colorsEnabled` removed).
- Playwright `viewport: null` + `--start-maximized` (chromium); per-scenario
  via the breakpoint registry.
- Scripts: `test`, `test:chromium`, `test:firefox`, `test:webkit`,
  `test:headed` (`HEADLESS=false`), `test:fast` (`SLOW_MO=0`),
  `generate-reports`.
- `worldParameters.video` (`mode`, `dir`, `size`, `filenamePattern`).
- `worldParameters.javascript` (`mode`, `levels`, `ignore`,
  `beforeScenario`, `afterScenario`).
- Deps: `@axe-core/playwright`, `axe-core`, `ajv`, `ajv-formats`,
  `js-yaml`, `tsx`.

`cat node_modules/webship-js/cucumber.js` to confirm before editing.

## Add a `worldParameters.users` registry (multi-role suites)

For CMS / multi-role apps (the Drupal pattern), seed
`worldParameters.users` at scaffold time:

```js
worldParameters: {
  launchUrl: process.env.LAUNCH_URL || '$target',
  users: {
    "Normal user":    { email: "test.authenticated@example.com", password: process.env.TEST_USER_PASSWORD },
    "Content editor": { email: "test.content_editor@example.com", password: process.env.TEST_USER_PASSWORD },
    "Content admin":  { email: "test.content_admin@example.com",  password: process.env.TEST_USER_PASSWORD },
    "Site admin":     { email: "test.site_admin@example.com",     password: process.env.TEST_USER_PASSWORD },
    "Super admin":    { email: "test.super_admin@example.com",    password: process.env.TEST_USER_PASSWORD }
  },
  // ...
}
```

Pair with a `Given I am a logged in user with the "<role>" user` custom
step (see `/webship-js-create` for the implementation pattern). This approach
collapses dozens of repeated login scenes into a single registry plus one
helper.

## Add a selector preset (optional but recommended)

Webship-js ships ready-made selector presets under
`node_modules/webship-js/tests/selectors/` for: Drupal, Drupal Core +
Claro, WordPress, Joomla, Magento 2, Ghost, Shopify, Strapi, TYPO3,
PrestaShop, Contentful, Craft, WooCommerce front, Bootstrap, Tailwind,
Material UI, Chakra, Bulma, Foundation, shadcn, Vuetify, Ant Design.

To use a preset, append to `cucumber.js`:

```js
selectors: {
  filesPath: './node_modules/webship-js/tests/selectors/',
  files: ['cms-drupal-cms-gin.json'],   // or your CMS
}
```

Then scenarios reference selectors by canonical name (`primary button`,
`notice success`, `modal close`, `data table`, `active tab`, etc.).

## Requirements

- Node.js >= 20.0 (enforced by webship-js `engines`).
- Linux: `npx playwright install --with-deps chromium` the first time
  (skip in DDEV — Dockerfile handles it).

## Verify

1. `cucumber.js` `require:` includes
   `node_modules/webship-js/tests/step-definitions/**/*.js` and
   `tests/step-definitions/**/*.js`.
2. `cucumber.js` uses `requireModule: ['tsx/cjs']`.
3. `package.json` has the six test scripts + `generate-reports`.
4. `tests/features/check-homepage.feature` runs green:
   `LAUNCH_URL=<url> npm run test:chromium`.

Report the project path, target URL, next-step command, and what scripts
were merged into `package.json`.

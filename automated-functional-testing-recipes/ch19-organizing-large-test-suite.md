# Chapter 19: Organizing a Large Test Suite

---

A test suite with five features is easy to manage. Put them all in one folder, run them all at once, done. But a mature application might have fifty features, three hundred scenarios, and twenty custom step definition files. Without a clear organizational strategy, you end up with a tangled mess where no one can find anything, tests run for hours, and new team members are afraid to add tests because they might break something.

This chapter shows you how to organize, tag, and scale a large webship-js test suite.

---

## File Organization Patterns

### Recipe 178: Domain-Based Directory Structure

**Problem:** It starts innocently: five feature files in one folder. A year later there are sixty, named things like `test2.feature` and `new-checkout-final.feature`, and the new hire's first PR adds a login scenario to a file that already had three -- because nobody could tell where login tests live. When a product owner asks "do we test password reset?", the honest answer is twenty minutes of grep. Structure is not bureaucracy; it is the difference between a suite people extend and a suite people fear.

**Solution:**

```
tests/
├── features/                           ← Feature files by domain
│   ├── auth/
│   │   ├── login.feature
│   │   ├── registration.feature
│   │   ├── password-reset.feature
│   │   └── logout.feature
│   ├── products/
│   │   ├── listing.feature
│   │   ├── search.feature
│   │   ├── cart.feature
│   │   └── checkout.feature
│   ├── admin/
│   │   ├── dashboard.feature
│   │   ├── users.feature
│   │   ├── content.feature
│   │   └── settings.feature
│   ├── api/
│   │   ├── users-api.feature
│   │   ├── products-api.feature
│   │   └── orders-api.feature
│   └── common/
│       ├── navigation.feature
│       ├── error-pages.feature
│       └── accessibility.feature
│
├── step-definitions/                    ← Step definitions by domain
│   ├── webship.js                       ← Built-in (from webship-js)
│   ├── webship-api.js                   ← Built-in API (from webship-js)
│   ├── auth-steps.js                    ← Login, logout, roles
│   ├── product-steps.js                 ← Cart, checkout, inventory
│   ├── admin-steps.js                   ← Admin panel operations
│   ├── viewport-steps.js               ← Responsive testing
│   └── a11y-steps.js                   ← Accessibility checks
│
├── support/                             ← Support files
│   ├── hooks.js                         ← Custom hooks
│   └── helpers.js                       ← Shared helper functions
│
└── assets/                              ← Test data files
    ├── test-image.png
    ├── test-document.pdf
    └── sample-csv.csv
```

**Rules:**
- Feature files are organized by **business domain** (auth, products, admin)
- Step definitions are organized by **domain**, not by keyword (Given/When/Then)
- Support files contain hooks and shared utilities
- Assets folder holds files used in upload tests

**How It Works:** The default cucumber.js profile loads every step file matching `tests/step-definitions/**/*.js` and every feature matching `tests/features/**/*.feature` -- recursively. That `**` is what makes this structure free: you can nest folders as deep as the domains demand and never touch configuration. The deeper win is that the directory tree doubles as a coverage map. `ls tests/features/auth/` answers "what do we test about authentication?" without opening a single file, and `npx cucumber-js tests/features/auth/` runs exactly that answer. Add `tests/selectors/` to the picture too -- named-selector JSON files auto-load from there via `worldParameters.selectors.filesPath` -- and a stranger can navigate the whole suite from the folder names alone.

**At scale:** Make this tree *identical* across every project you maintain. When all twenty suites share the same skeleton -- same folder names, same domains where they apply -- a QA engineer rotating from project A to project B knows where everything lives before opening the repo, and your scaffolding scripts, CI templates, and onboarding docs are written once. At Vardot this was the quiet rule that made Varbase testing survivable: the structure was boring on purpose, and boring scales.

---

### Recipe 179: Step Definition Organization

**Problem:** Someone needs to fix the logout step. They open `when-steps.js` -- 300 lines of unrelated actions sorted by keyword -- and scroll past cart steps, admin steps, and viewport steps hunting for it. Worse, the matching assertions live in a different file, so understanding "how do we test auth?" means reading three files in parallel. Keyword-sorted step files are the junk drawer of test suites: everything is technically somewhere.

**Solution:**

Split step definitions by domain, keeping related steps together:

```javascript
// tests/step-definitions/auth-steps.js
// All authentication-related steps in one file

const { Given, When, Then } = require('@cucumber/cucumber');

// Users configuration
const users = {
  admin: { email: 'admin@example.com', password: 'AdminPass123' },
  editor: { email: 'editor@example.com', password: 'EditorPass123' },
  customer: { email: 'customer@example.com', password: 'CustPass123' },
};

Given('I am a logged in user with the {string} user', async function (role) {
  const user = users[role];
  await this.page.goto(this.launchUrl + '/login');
  await this.page.waitForLoadState('networkidle');
  await this.page.getByLabel('Email').fill(user.email);
  await this.page.getByLabel('Password').fill(user.password);
  await this.page.getByRole('button', { name: 'Sign In' }).click();
  await this.page.waitForLoadState('networkidle');
});

When('I log out', async function () {
  await this.page.locator('a, button').filter({ hasText: 'Logout' }).first().click();
  await this.page.waitForLoadState('networkidle');
});
```

```javascript
// tests/step-definitions/product-steps.js
// All product/cart-related steps

const { When, Then } = require('@cucumber/cucumber');
const assert = require('assert');

When('I add {string} to the cart', async function (product) { ... });
When('I remove {string} from the cart', async function (product) { ... });
Then('the cart should have {int} items', async function (count) { ... });
Then('the total should be {string}', async function (total) { ... });
```

**Avoid organizing by keyword** (all Given steps in one file, all When steps in another). That makes it impossible to understand a domain at a glance:

```
BAD:                              GOOD:
given-steps.js  ← 200 lines     auth-steps.js      ← focused
when-steps.js   ← 300 lines     product-steps.js   ← focused
then-steps.js   ← 200 lines     admin-steps.js     ← focused
```

**How It Works:** Cucumber does not care how you split the files -- all step definitions land in one global registry at load time, and any feature can use any step. The organization is purely for humans, which is exactly why it matters: a domain file is a self-contained story (here is how we log in, here is how we log out, here is the user data both share), while a keyword file is a concordance. Keeping the `users` map at the top of `auth-steps.js` is the same principle at file level -- the data lives next to the only steps that use it. One rule the global registry enforces for you: a step pattern must be unique across *all* files, so a duplicate `When I log out` in two domain files is a load-time ambiguity error, not a silent override. Take it as a feature -- it forces the conversation about which domain owns the step. webship-js follows this layout itself: 411 steps across 36 domain-focused files (forms, navigation, api, cookies, clock, iframe...), not three keyword buckets.

**At scale:** The domain split is also your extraction roadmap. When `auth-steps.js` looks the same in your fourth project, that file wants to become a shared internal package -- one version, one fix, twenty consumers. Files organized by keyword never reveal this; files organized by domain make the duplication impossible to miss.

---

## Tag Strategies

### Recipe 180: Tag Naming Conventions

**Problem:** Six months in, your features carry `@smoke`, `@Smoke`, `@quick`, `@fast`, and `@critical-path` -- five tags, one meaning, invented by five people who never agreed on a vocabulary. Then the release pipeline runs `--tags "@smoke"` and silently skips the scenarios tagged `@Smoke`, and nobody notices until a broken login ships. Tags are an API between your features and your pipelines; an API needs a published contract.

**Solution:**

```gherkin
# Severity tags
@smoke           # Critical path tests (run on every PR)
@regression      # Full regression suite (run nightly)
@full            # Everything including edge cases

# Feature area tags
@auth            # Authentication features
@products        # Product-related features
@admin           # Admin panel features
@api             # API tests (no browser)

# Environment tags
@local           # Only run locally
@staging         # Only run on staging
@production      # Safe for production

# Status tags
@wip             # Work in progress (skip in CI)
@skip            # Temporarily skipped (with reason in description)
@flaky           # Known flaky test (quarantined)

# Browser tags
@chromium        # Chromium-specific
@skip-webkit     # Skip on WebKit

# Type tags
@a11y            # Accessibility tests
@visual          # Visual regression tests
@performance     # Performance-related tests
```

**How It Works:** Tags are case-sensitive strings matched literally by Cucumber's tag-expression engine -- there is no fuzzy matching and no warning for a tag that nothing selects, which is why the convention has to live in a document, not in tribal memory. Keep the dimensions orthogonal: one severity tag, one feature-area tag, plus status tags as needed, so `@smoke @auth` composes cleanly in expressions like `--tags "@smoke and @auth"`. Reserve a namespace for the tags webship-js itself interprets -- `@screenshots`, `@video`, `@no-video`, `@js-fail`, `@js-warn`, `@js-off` -- these are not labels, they change runtime behavior, and a teammate inventing `@video` to mean "tests the video player" will have a confusing afternoon. And every `@skip` or `@flaky` should carry a reason and a ticket reference in the scenario description; a quarantine tag without an owner is how flaky tests become permanent residents.

**At scale:** This is the single highest-leverage standardization for a multi-project team. One tag vocabulary, versioned in a shared document, adopted by every suite -- because your CI templates hardcode `--tags "@smoke"`, your dashboards group by `@critical` pass rate per project, and your PMO reads those dashboards to answer "is release X safe?". A project that renames `@smoke` to `@quick` does not just break its own pipeline; it falls off every cross-project report.

---

### Recipe 181: Test Suite Tiers

**Problem:** Your full suite takes 45 minutes. Run it on every pull request and developers stop waiting for green -- they merge on faith, or they stop writing tests because tests are "what makes CI slow". Run only a hand-picked few and a regression slips through the gap. The way out is not one suite but a ladder: a two-minute answer for every PR, a deeper answer every night, and the full answer before a release.

**Solution:**

```bash
# Tier 1: Smoke Tests (run on every PR, ~2 minutes)
npx cucumber-js --tags "@smoke"

# Tier 2: Regression (run nightly, ~15 minutes)
npx cucumber-js --tags "@smoke or @regression"

# Tier 3: Full Suite (run on release, ~45 minutes)
npx cucumber-js --tags "not @skip and not @wip"

# Run specific domain
npx cucumber-js --tags "@auth"
npx cucumber-js --tags "@api"

# Exclude flaky tests
npx cucumber-js --tags "not @flaky"

# Complex tag expressions
npx cucumber-js --tags "@smoke and not @flaky"
npx cucumber-js --tags "(@auth or @products) and not @wip"
```

```
Test Suite Tiers:
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Tier 1: SMOKE                 (~10 scenarios, ~2 min)           │
│  ┌──────────────────┐                                           │
│  │ Login works      │  Run on: Every PR                         │
│  │ Homepage loads   │  Purpose: Catch catastrophic breakage     │
│  │ API health check │                                           │
│  └──────────────────┘                                           │
│                                                                  │
│  Tier 2: REGRESSION            (~50 scenarios, ~15 min)          │
│  ┌──────────────────┐                                           │
│  │ All of Tier 1    │  Run on: Nightly + pre-merge              │
│  │ CRUD operations  │  Purpose: Catch functional regressions    │
│  │ Role-based access│                                           │
│  │ Form validation  │                                           │
│  └──────────────────┘                                           │
│                                                                  │
│  Tier 3: FULL                  (~200 scenarios, ~45 min)         │
│  ┌──────────────────┐                                           │
│  │ All of Tier 2    │  Run on: Release candidate                │
│  │ Edge cases       │  Purpose: Maximum coverage before deploy  │
│  │ Multi-browser    │                                           │
│  │ Accessibility    │                                           │
│  │ Visual regression│                                           │
│  └──────────────────┘                                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** The tiers are nothing but tag expressions -- the same suite, filtered three ways. `@smoke or @regression` makes the tiers cumulative by construction, so a scenario promoted to smoke is never accidentally dropped from the nightly. `not @skip and not @wip` for the release tier flips the logic: instead of opting scenarios *in*, it runs everything that has not been explicitly benched, which is the right default when the stakes are highest. Discipline lives at the boundaries. Smoke membership should be a review decision with a budget -- "smoke stays under three minutes" is a rule someone has to enforce, because every author believes their scenario is critical, and a fifteen-minute smoke tier is just a slow regression tier with a misleading name. Each tier writes the same `tests/reports/cucumber_report.json`, so one report pipeline serves all three.

**At scale:** Standardize the tier *names and meanings* across projects, not the scenario counts -- a small brochure site's regression tier might be eight scenarios, a commerce platform's might be three hundred, and that is fine. What must match is the contract: every project answers "smoke" in minutes on every PR and "full" before each release. That symmetry lets DevOps build one pipeline template with the tag expression as a parameter, and lets the PMO read "smoke green, regression green" identically across the whole portfolio. It is also your CI budget knob: nightly cost scales with the regression tier, so when the farm bill climbs, you tune tags rather than re-architect pipelines.

---

### Recipe 182: Environment-Specific Tags

**Problem:** A test that resets the database is a feature on localhost and a disaster on production. A payment-gateway test only makes sense where the sandbox gateway is wired up. The night someone points the full suite at the wrong base URL -- and someone eventually will -- the only thing standing between "oops" and "incident report" is whether your scenarios know which environments they are allowed to touch.

**Solution:**

```gherkin
@local
Feature: Development-Only Tests
  Scenario: Debug endpoint returns data
    Given the API base URL is "http://localhost:8080"
    When I send a GET request to "/api/debug/info"
    Then the API response code should be 200

@staging
Feature: Staging Integration
  Scenario: Third-party payment gateway works
    Given I am on "https://staging.example.com"
    ...

@production @readonly
Feature: Production Health Check
  Scenario: Homepage loads
    Given I am on "https://www.example.com"
    Then I should see "Welcome"
```

```bash
# Local development
npx cucumber-js --tags "@local or (not @staging and not @production)"

# Staging deployment
npx cucumber-js --tags "@staging or @smoke"

# Production health check (read-only tests only)
npx cucumber-js --tags "@production and @readonly"
```

**How It Works:** Two mechanisms cooperate here. The tag expression decides *which scenarios run*; the `LAUNCH_URL` environment variable decides *where they run* -- it feeds `worldParameters.launchUrl`, which every relative-path step (`Given I am on "/path"`) resolves against. Keep them paired in your pipeline definitions: the job that sets `LAUNCH_URL` to production is the same job that passes `--tags "@production and @readonly"`, so the safe filter cannot be forgotten separately from the dangerous URL. The `@production and @readonly` double-tag is deliberate defense in depth -- a scenario must be explicitly marked safe twice before it touches the live site. And prefer relative paths in scenarios over the hardcoded URLs shown in the staging example above; a scenario that bakes in `staging.example.com` can never be reused as a production health check.

**At scale:** Environment wiring is where DevOps and QA meet, and the clean handshake is: QA owns the tags, DevOps owns the variables. CI injects `LAUNCH_URL`, credentials, and `BROWSER` per environment from pipeline secrets; the repository contains zero environment-specific URLs. Do this on all your projects and the production smoke check becomes a one-template cron job -- the same `--tags "@production and @readonly"` line, twenty different `LAUNCH_URL` values, one wake-up call when any of them goes red.

---

## Scaling the Suite

### Recipe 183: Parallel Execution

**Problem:** The nightly regression takes fifty minutes, the team keeps adding scenarios, and the morning standup starts before the run finishes. Sequential execution means your feedback loop grows linearly with your coverage -- the better your testing gets, the slower your answers arrive. Most of that time the CPU is idle, waiting on page loads. Parallel workers turn that waiting into throughput.

**Solution:**

Cucumber-js supports parallel execution:

```bash
# Run with 4 parallel workers
npx cucumber-js --parallel 4

# Run with workers matching CPU cores
npx cucumber-js --parallel $(nproc)
```

```
Sequential:                      Parallel (4 workers):
┌──────┐                        ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ S1   │                        │ S1   │ │ S2   │ │ S3   │ │ S4   │
│ S2   │                        │ S5   │ │ S6   │ │ S7   │ │ S8   │
│ S3   │ Total: 20 min          │ S9   │ │ S10  │ │      │ │      │
│ S4   │                        └──────┘ └──────┘ └──────┘ └──────┘
│ ...  │                        Total: ~5 min
│ S10  │
└──────┘
```

**Requirements for parallel execution:**
- Each scenario must be fully independent (no shared state)
- Database operations should not conflict between workers
- File paths should be unique per worker (for screenshots, reports)
- webship-js already ensures this: each scenario gets its own browser

**How It Works:** `--parallel N` spawns N worker processes, and Cucumber hands each worker the next pending scenario as it frees up -- dynamic distribution, so one slow feature does not strand a worker while others sit idle. Each worker is a full process with its own browser, its own World, and its own `BeforeAll`/`AfterAll` (remember Recipe 176: "once" means once per worker). Browser isolation comes free from the webship-js lifecycle, which leaves *application state* as your real risk: two workers registering `test-user@example.com` at the same moment will collide in the database even though their browsers never met. Make test data unique per scenario -- a timestamp or random suffix in usernames is usually enough. Speedup is sublinear: browsers are memory-hungry, so on a typical 4-core CI runner, 3-4 workers is the sweet spot before swapping eats your gains.

**Pro Tip:** When a scenario fails only under `--parallel`, do not tag it `@flaky` and move on -- it is almost always a shared-state bug telling you two scenarios touch the same record. Fix the data collision and you have made the suite safer everywhere, not just faster.

---

### Recipe 184: Sharding Tests Across CI Workers

**Problem:** You have maxed out `--parallel` on one runner -- the machine is out of cores and memory before you are out of scenarios, and the release-candidate run still takes forty minutes. The next ceiling to break is the machine itself: split the feature files across four CI machines that run simultaneously, and the wall-clock time becomes whatever the slowest shard takes.

**Solution:**

```yaml
# GitHub Actions matrix strategy
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - name: Run shard ${{ matrix.shard }} of 4
        run: |
          # Get all feature files
          FEATURES=$(find tests/features -name "*.feature" | sort)
          TOTAL=$(echo "$FEATURES" | wc -l)
          PER_SHARD=$(( (TOTAL + 3) / 4 ))  # Ceiling division
          START=$(( (${{ matrix.shard }} - 1) * PER_SHARD + 1 ))
          SHARD_FEATURES=$(echo "$FEATURES" | sed -n "${START},$((START + PER_SHARD - 1))p")
          npx cucumber-js $SHARD_FEATURES
```

**How It Works:** The matrix strategy launches four identical jobs that differ only in the `shard` variable. Each job lists all feature files, sorts them -- the `sort` is essential, because the split is only deterministic if every machine sees the same order -- and takes its quarter by line range. There is no coordinator and no shared state between machines, which is what makes this trivially portable to GitLab CI, Azure Pipelines, or any of the other CI configs webship-js ships. The trade-off versus `--parallel` is balance: sharding splits by *file count*, not by runtime, so a shard holding your three heaviest features finishes last while others idle. Two mitigations: combine both levels (`--parallel 2` inside each shard smooths the imbalance), and if one feature file dominates, split it into smaller files -- which usually improves organization anyway. Each shard writes its own `cucumber_report.json`, so add a final job that collects the four artifacts and merges them before report generation, or you will have four partial answers and no single verdict.

**At scale:** Sharding multiplies cost visibly: four machines for ten minutes is the same compute bill as one for forty, plus four browser-image pulls and four `npm install`s of overhead. Reserve full sharding for the suites and the moments that need it -- release candidates, the two or three heaviest products -- and let small projects run unsharded. And pin the browser image version across all shards in the template; a shard farm where machine 3 pulled a newer Chromium than the others produces the most confusing "flaky" failures you will ever debug.

---

### Recipe 185: Running Specific Features

**Problem:** You are iterating on one failing scenario, and every check costs a full-suite run -- twenty minutes to learn whether a one-line selector fix worked. That feedback loop trains people to push and pray instead of verifying locally. The fix is surgical targeting: run exactly the file, folder, scenario, or tag you are working on, and save the full run for CI.

**Solution:**

```bash
# Single feature file
npx cucumber-js tests/features/auth/login.feature

# All features in a directory
npx cucumber-js tests/features/auth/

# Multiple specific files
npx cucumber-js tests/features/auth/login.feature tests/features/products/cart.feature

# By tags
npx cucumber-js --tags "@smoke"
npx cucumber-js --tags "@auth and not @wip"

# Specific scenario by line number
npx cucumber-js tests/features/auth/login.feature:15
```

**How It Works:** Path arguments *replace* the default feature glob from cucumber.js, while everything else in the profile -- step definitions, the 45s timeout, the JSON report target -- still loads, so a targeted run behaves identically to the full suite, just smaller. The `file.feature:15` form runs the single scenario whose declaration spans that line; failed-run output prints these `path:line` references precisely so you can paste one back and re-run only what broke. Combine dimensions freely: `npx cucumber-js tests/features/auth/ --tags "not @wip"` intersects a folder with a tag filter. The debugging trifecta on a single scenario is `HEADLESS=false npx cucumber-js tests/features/auth/login.feature:15` -- a visible browser (with the headed-mode 800ms SLOW_MO making each action watchable) on exactly the failing case, seconds after CI told you the line number.

---

## Chapter Summary

- Organize features by business domain (auth/, products/, admin/)
- Organize step definitions by domain, not by keyword
- Use a consistent tag naming convention across the team
- Implement test suite tiers: smoke (fast), regression (thorough), full (comprehensive)
- Environment tags prevent running dangerous tests in production
- Parallel execution with `--parallel N` speeds up long suites
- CI sharding distributes tests across machines for maximum speed
- Each scenario must be independent for parallel/shard execution to work
- Run specific files, folders, or tagged subsets as needed

Next: Design Patterns for Tests -- proven patterns for maintainable test code.

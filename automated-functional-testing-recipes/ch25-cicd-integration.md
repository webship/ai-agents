# Chapter 25: CI/CD Integration

---

A test suite that only runs on your laptop is a hobby project. A test suite that runs automatically on every push, every pull request, and every deployment is a production-grade quality gate. CI/CD integration transforms your tests from "something I run sometimes" to "something that protects the team always."

This chapter provides copy-paste CI configurations for the platforms you are most likely to meet. You will not be starting from scratch either way: the webship-js repo ships ready-made configs for thirteen CI providers -- GitHub Actions, GitLab CI, Bitbucket Pipelines, Azure Pipelines, CircleCI, Travis, Jenkins, Drone, Semaphore, AWS CodeBuild, Google Cloud Build, Codefresh, and Bamboo. The recipes here explain the moving parts those configs share, because the day a pipeline fails, you need to know which part moved.

---

## CI/CD Pipeline with Testing Gates

<!-- diagram: cicd-pipeline -->

```
┌──────────────────────────────────────────────────────────────────┐
│ CI/CD Pipeline                                                   │
│                                                                  │
│  Developer pushes code                                           │
│        │                                                         │
│        ▼                                                         │
│  ┌──────────────┐                                               │
│  │ Lint & Build  │ ← Fast checks (seconds)                     │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Unit Tests   │ ← Fast (seconds to minutes)                  │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Smoke Tests  │ ← webship-js @smoke (2-5 min)                │
│  │ (Chromium)   │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ PR Approved  │ ← Human review                               │
│  │ Merge to main│                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────┐               │
│  │ Full Regression (all browsers, parallel)     │               │
│  │ ┌────────┐ ┌────────┐ ┌────────┐           │               │
│  │ │Chromium│ │Firefox │ │WebKit  │           │               │
│  │ └────────┘ └────────┘ └────────┘           │               │
│  └──────┬───────────────────────────────────────┘               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Deploy to    │ ← Only if all tests pass                     │
│  │ staging/prod │                                               │
│  └──────────────┘                                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Recipe 220: GitHub Actions Setup

**Problem:** The suite is green on every laptop, and that is exactly the problem -- it only runs when someone remembers to run it. A teammate merges late on a Thursday without running the tests, and Friday's standup opens with "who broke login?" Nobody broke it on purpose; there was simply no gate. You need the suite to run on every push and every pull request, with the results attached to the PR where the reviewer is already looking.

**Solution:**

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  smoke-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Start application
        run: npm run start &
        env:
          NODE_ENV: test

      - name: Wait for application
        run: npx wait-on http://localhost:8080 --timeout 30000

      - name: Run smoke tests
        run: BROWSER=chromium SLOW_MO=0 npx cucumber-js --tags "@smoke"
        env:
          LAUNCH_URL: http://localhost:8080

      - name: Upload screenshots on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshots
          path: screenshots/

  full-regression:
    needs: smoke-test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps ${{ matrix.browser }}

      - name: Start application
        run: npm run start &
        env:
          NODE_ENV: test

      - name: Wait for application
        run: npx wait-on http://localhost:8080 --timeout 30000

      - name: Run tests on ${{ matrix.browser }}
        run: BROWSER=${{ matrix.browser }} SLOW_MO=0 npx cucumber-js
        env:
          LAUNCH_URL: http://localhost:8080

      - name: Upload test artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.browser }}
          path: |
            screenshots/
            reports/
```

**How It Works:** The workflow defines two jobs with different speed budgets. `smoke-test` is the PR gate: it installs dependencies with `npm ci` (lockfile-exact, so CI runs the same dependency tree that passed on a laptop), installs Chromium plus its OS libraries with `--with-deps`, boots the app, and runs only `@smoke` scenarios on a single browser. `full-regression` runs after it, fanned out by the matrix into three independent runners -- one per browser engine. The wiring between CI and webship-js is all environment variables, and that is deliberate. `BROWSER` is read by webship-js's `playwright.config.ts` to pick chromium, firefox, or webkit. `LAUNCH_URL` feeds `worldParameters.launchUrl`, so feature files never hardcode a host. `SLOW_MO=0` is technically redundant -- webship-js already defaults slow-motion to 0 in CI -- but stating it makes the config self-documenting. The `wait-on` step matters more than it looks: `npm run start &` returns immediately, and without the wait your first scenario races the app boot and dies with a connection error that has nothing to do with your tests. Finally, the screenshot upload runs `if: failure()` -- webship-js captures a screenshot of every failed step on its own (`WEBSHIP_SCREENSHOT_ON_FAILED` defaults to true, files land in `./screenshots` with a `failed_` prefix), so the artifact holds exactly what you need on a red run and the step is skipped on green ones.

**At scale:** When you own twenty repos, do not paste this file twenty times. Extract it into a reusable workflow (`workflow_call`) in a shared repo and have each project call it with two inputs: the launch URL and the tag expression. A pipeline fix then ships to the whole fleet in one PR instead of twenty. Wire `LAUNCH_URL` through repository or environment variables instead of editing YAML per project, and budget the matrix consciously -- three browsers times forty projects is 120 runner jobs a night. Most teams keep the PR gate on Chromium-only `@smoke` and spend the multi-browser minutes on a nightly schedule.

**Pro Tip:** Add `WEBSHIP_VIDEO: on-failure` to the regression job's `env` and include `videos/` in the artifact paths. A video of the failing scenario answers "what actually happened on the runner?" faster than any log -- and green runs keep nothing, so artifact storage stays flat.

---

## Recipe 221: GitLab CI Setup

**Problem:** The pipeline worked for months, then a routine `npm update` bumps Playwright and every job starts failing with "Executable doesn't exist" -- the runner image still carries last quarter's browsers. Or it is day one: tests pass locally and die in CI with missing system libraries, and you lose an afternoon to apt-get archaeology. Both are the same disease. The browser environment in CI is not the one your dependency tree expects.

**Solution:**

```yaml
# .gitlab-ci.yml
image: mcr.microsoft.com/playwright:v1.58.0-jammy

stages:
  - test

variables:
  LAUNCH_URL: http://localhost:8080
  SLOW_MO: "0"

smoke-test:
  stage: test
  script:
    - npm ci
    - npm run start &
    - npx wait-on $LAUNCH_URL --timeout 30000
    - BROWSER=chromium npx cucumber-js --tags "@smoke"
  artifacts:
    when: on_failure
    paths:
      - screenshots/

full-regression:
  stage: test
  parallel:
    matrix:
      - BROWSER: [chromium, firefox, webkit]
  script:
    - npm ci
    - npm run start &
    - npx wait-on $LAUNCH_URL --timeout 30000
    - npx cucumber-js
  artifacts:
    when: always
    paths:
      - screenshots/
      - reports/
```

**How It Works:** The whole recipe hangs on the first line. `mcr.microsoft.com/playwright:v1.58.0-jammy` is Microsoft's official image with Chromium, Firefox, and WebKit preinstalled, along with every OS library they need -- which is why there is no `playwright install` step anywhere in this file. The rule that keeps it working: the image tag must match the Playwright version in your `package.json`. Playwright looks for browser builds keyed to its own version, so if the lockfile says 1.59 and the image says v1.58.0, the builds it wants are simply not there -- and you are back to "Executable doesn't exist". Treat the image tag and the dependency as one thing; upgrade them in the same merge request. The rest is wiring: `variables` exports `LAUNCH_URL` and `SLOW_MO` to every job, `parallel: matrix` expands `full-regression` into three jobs that each receive a different `BROWSER` value for webship-js's `playwright.config.ts` to read, and the two artifacts blocks differ on purpose -- `when: on_failure` keeps smoke artifacts lean, while `when: always` preserves regression reports even on green runs so you have a baseline to compare against.

**At scale:** Forty projects means forty `.gitlab-ci.yml` files quietly pinning forty image tags -- and they will drift. Move the image line and the job templates into a shared CI include (`include: project: qa/ci-templates`) and let each project's file shrink to a few variables. The next Playwright upgrade becomes one MR in one repo, and the fleet follows. It also gives your platform team a single answer to "which browser versions are we actually testing with?" -- a question that is otherwise forty answers.

---

## Recipe 222: Jenkins Pipeline Setup

**Problem:** Your Jenkins agents are snowflakes. Someone installed Chrome on agent-03 by hand two years ago, agent-07 has a different glibc, and the same suite passes or fails depending on which agent grabs the build. Nobody remembers how the working agent was set up, so nobody dares touch it. You need builds that carry their browser environment with them.

**Solution:**

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'mcr.microsoft.com/playwright:v1.58.0-jammy'
        }
    }

    environment {
        LAUNCH_URL = 'http://localhost:8080'
        SLOW_MO = '0'
    }

    stages {
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Smoke Tests') {
            steps {
                sh 'npm run start &'
                sh 'npx wait-on $LAUNCH_URL --timeout 30000'
                sh 'BROWSER=chromium npx cucumber-js --tags "@smoke"'
            }
        }

        stage('Full Regression') {
            parallel {
                stage('Chromium') {
                    steps {
                        sh 'BROWSER=chromium npx cucumber-js'
                    }
                }
                stage('Firefox') {
                    steps {
                        sh 'BROWSER=firefox npx cucumber-js'
                    }
                }
                stage('WebKit') {
                    steps {
                        sh 'BROWSER=webkit npx cucumber-js'
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'screenshots/**,reports/**', allowEmptyArchive: true
        }
    }
}
```

**How It Works:** `agent { docker { image '...' } }` is the snowflake killer: every build runs inside the pinned Playwright container no matter which physical agent picks it up, so "works on agent-03" stops being a sentence anyone says. The same version rule from the GitLab recipe applies -- the image tag and your `package.json` Playwright version move together. The `environment` block exports `LAUNCH_URL` and `SLOW_MO` to every stage. One thing to watch: the three regression stages run in `parallel` inside the same container, which means three browsers share one agent's CPU. On a two-core agent they starve each other, scenarios stretch, and you can blow through the 45-second step timeout that webship-js's default Cucumber profile sets -- failures that look like app bugs but are really contention. If you see that, run the browsers sequentially or give the job a bigger agent. The `post { always }` block archives screenshots and reports before the container is torn down; without it, the evidence vanishes with the workspace.

**Pro Tip:** If the app you started with `npm run start &` is mysteriously dead by the regression stage, you have met Jenkins's ProcessTreeKiller, which reaps background processes when a step ends. Start the app with `JENKINS_NODE_COOKIE=dontKillMe npm run start &` to keep it alive across stages.

---

## Recipe 223: Test Reports in CI

**Problem:** The nightly run went red and the only record is a four-thousand-line console log on a CI page half the team cannot even access. The delivery manager asks "what failed, and does it block the release?" -- and producing an honest answer takes you twenty minutes of scrolling. Results that exist only as terminal output are results only one person can read, and that person becomes the bottleneck for every release question.

**Solution:**

```javascript
// cucumber.js - CI profile with JSON and HTML reporting
module.exports = {
  ci: {
    format: [
      '@cucumber/pretty-formatter',
      'json:reports/cucumber-report.json',
    ],
    // ... other config
  },
};
```

Generate HTML report after tests:

```javascript
// generate-report.js
const reporter = require('cucumber-html-reporter');

reporter.generate({
  theme: 'bootstrap',
  jsonFile: 'reports/cucumber-report.json',
  output: 'reports/cucumber-report.html',
  reportSuiteAsScenarios: true,
  launchReport: false,
  metadata: {
    'Browser': process.env.BROWSER || 'chromium',
    'Platform': process.platform,
    'Environment': process.env.NODE_ENV || 'test',
  },
});
```

**How It Works:** Two formats, two audiences. The `ci` profile keeps the pretty formatter for humans watching the console and adds `json:reports/cucumber-report.json` for machines -- a complete record of every scenario, step, status, duration, and attachment. `cucumber-html-reporter` then renders that JSON into a self-contained HTML page where a red scenario is a red row anyone can click open. The `metadata` block is not decoration: stamping browser, platform, and environment onto the report is what lets you tell "failed on webkit against staging" from "failed on chromium against prod" three weeks later. And webship-js ships this whole pipeline ready-made -- `bin/generate-reports.js` wraps the same reporter with four themes (bootstrap, hierarchy, foundation, simple), can emit PDF as well as HTML (`--format html|pdf|all`, plus `--pdf-format` and `--pdf-landscape`), and takes metadata as flags: `--app-version`, `--env`, `--browser`, `--platform`, `--parallel`, `--executed`, and any number of custom `--metadata Key=Value` pairs. In CI you can pass extra flags through the `WEBSHIP_REPORT_ARGS` environment variable, or set `WEBSHIP_REPORT_DISABLE` for runs where you genuinely do not want a report generated.

**At scale:** The JSON report schema is identical for every project, and that is the lever. Stamp each run with `--app-version`, `--env`, and a `--metadata Project=name` pair, collect the JSON files from all forty nightly runs into one place, and a small script can answer "which projects went red last night, on which browsers, since which app version" -- the cross-project quality signal your PMO keeps asking for and nobody can currently produce. The PDF output earns its keep here too: a generated, versioned PDF attached to the release ticket is a sign-off artifact a non-technical stakeholder can actually open.

---

## Recipe 224: Deployment Gate Testing

**Problem:** It is 4:45 on a Friday and the deploy job runs anyway -- red suite, green deploy, because nothing in the pipeline actually connects them. The hotfix happens at midnight, and the retrospective writes itself: tests that do not gate anything are advisory, and advisory quality checks lose every argument with a deadline. The gate has to be structural, not a team agreement to "check the tests first".

**Solution (GitHub Actions):**

```yaml
# Branch protection rule: Require "smoke-test" job to pass
# Settings → Branches → Branch protection rules → Require status checks

# The workflow ensures tests must pass before merge:
jobs:
  smoke-test:
    # ... (must pass for PR merge)

  deploy:
    needs: [smoke-test, full-regression]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: echo "Deploying..."
        # Only runs if ALL test jobs passed
```

**How It Works:** There are two separate gates here, and you want both. The merge gate is the branch protection rule: marking `smoke-test` as a required status check greys out the merge button until it passes, so red code cannot reach `main` by accident -- not even with an approving review. The deploy gate is the `needs: [smoke-test, full-regression]` line: GitHub Actions builds a dependency graph from `needs`, and if any listed job fails, the deploy job is skipped -- not failed, skipped, which reads honestly in the run timeline. The `if: github.ref == 'refs/heads/main'` condition keeps deploys off feature branches even though the same workflow runs there. For production you can add a third, human gate: a GitHub environment with required reviewers, so the pipeline proves the build is safe and a person still decides when it ships.

**At scale:** Use the same job names -- `smoke-test`, `full-regression`, `deploy` -- in every repository, because the names are the interface. A release manager who can read one project's checks can now read all forty, and "is release X safe?" stops being a meeting and becomes a glance at the gate. Pair that with the report artifact from Recipe 223 attached to each release and you have an evidence trail: what ran, where, against which version, and what the gate saw when it opened.

---

## Chapter Summary

- CI/CD transforms tests from manual to automatic
- Smoke tests on every PR (fast feedback, 2-5 minutes)
- Full regression on merge/nightly (comprehensive, all browsers)
- Upload screenshots as artifacts for debugging CI failures
- Use matrix strategies for parallel multi-browser testing
- Generate HTML reports for team visibility
- Deployment gates prevent broken code from reaching production
- Playwright Docker images include all browser dependencies
- Pin the Playwright image tag to the Playwright version in package.json -- upgrade them together
- Stamp reports with version, environment, and browser metadata so runs stay comparable across projects

Next: Testing CMS Applications -- patterns specific to content management systems.

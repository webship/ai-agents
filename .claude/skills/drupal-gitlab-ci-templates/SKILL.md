---
name: drupal-gitlab-ci-templates
description: Author and configure a Drupal project's .gitlab-ci.yml on git.drupalcode.org using the official drupal/gitlab_templates — the include block, the build/validate/test stages, every job (composer, phpcs, phpstan, cspell, eslint, stylelint, phpunit, nightwatch, upgrade status, test-only changes), the customization variables, core-version variants, custom browser-test jobs, the runner time limit and parallel jobs, and how to skip, pin or make a job required. Use when adding CI to a module, theme, profile or recipe, when a pipeline job fails, times out or is missing, when choosing which jobs must gate a merge, or when pinning the templates ref or the Drupal core version.
---

# Drupal GitLab CI templates

The official CI for drupal.org projects is **[`drupal/gitlab_templates`](https://www.drupal.org/project/gitlab_templates)**
(docs: <https://project.pages.drupalcode.org/gitlab_templates/>). Do not hand-roll a pipeline for a
contrib project — include the template and customize it.

To run any of it on your own machine first, use the **`drupal-gitlab-ci-local-runner`** skill.

## 1. The include block

`.gitlab-ci.yml` at the project root:

```yaml
include:
  - project: $_GITLAB_TEMPLATES_REPO
    ref: $_GITLAB_TEMPLATES_REF
    file:
      - /includes/include.drupalci.main.yml
      - /includes/include.drupalci.variables.yml
      - /includes/include.drupalci.workflows.yml
```

The remote form is equivalent and is what the upstream docs show:

```yaml
include:
  - remote: https://git.drupalcode.org/${_GITLAB_TEMPLATES_REPO}/-/raw/${_GITLAB_TEMPLATES_REF}/includes/include.drupalci.main.yml
  - remote: https://git.drupalcode.org/${_GITLAB_TEMPLATES_REPO}/-/raw/${_GITLAB_TEMPLATES_REF}/includes/include.drupalci.variables.yml
  - remote: https://git.drupalcode.org/${_GITLAB_TEMPLATES_REPO}/-/raw/${_GITLAB_TEMPLATES_REF}/includes/include.drupalci.hidden-variables.yml
  - remote: https://git.drupalcode.org/${_GITLAB_TEMPLATES_REPO}/-/raw/${_GITLAB_TEMPLATES_REF}/includes/include.drupalci.workflows.yml
```

Two variables drive it:

| Variable | Value |
|---|---|
| `_GITLAB_TEMPLATES_REPO` | `project/gitlab_templates` |
| `_GITLAB_TEMPLATES_REF` | the templates ref to pin — a tag, or `main` |

**Pin the ref.** Tracking `main` means an upstream change can turn a green project red overnight with
no commit of yours. Pin a tag, and bump it deliberately as its own issue + MR.

## 2. Stages and jobs

| Family | Jobs |
|---|---|
| **Build** | `composer`, `drupal cms`, `pages` |
| **Validate** | `composer-lint`, `cspell`, `eslint`, `phpcs`, `phpstan`, `stylelint`, secret detection |
| **Test** | `phpunit`, `nightwatch`, `test-only changes`, `upgrade status` |

`composer` runs first and every later job consumes its artifact — a failing `composer` job fails
everything downstream, so read that job's log first when a pipeline collapses.

## 3. Customization variables

Set these under a top-level `variables:` block, or scoped to one job.

| Variable | What it does |
|---|---|
| `PROJECT_NAME` | Override the auto-detected project name |
| `PROJECT_TYPE` | `module`, `theme` or `profile` |
| `DRUPAL_CORE` | The core version to build against |
| `DRUPAL_PROJECTS_PATH` | Install path (defaults per type, e.g. `modules/custom`) |
| `DRUPAL_RECIPES_PATH` | Recipe directory |
| `_NODE_VERSION` | Node version, overriding `.nvmrc` |
| `_TARGET_PHP` | The PHP version of the jobs |
| `OPT_IN_TEST_NEXT_MINOR` / `OPT_IN_TEST_NEXT_MAJOR` | Also test against core's next minor / major |
| `_SHOW_ENVIRONMENT_VARIABLES` | Print the environment in the job log — useful once, noisy forever |
| `SKIP_<JOB>` | `SKIP_PHPSTAN: 1`, `SKIP_ESLINT: 1`, … disables that job |

A value entered in the pipeline UI form beats the file.

### Change the core version

Scope it to the job variant, not globally:

```yaml
composer:
  variables:
    DRUPAL_CORE: 11.4.8
```

When your `composer.json` pins a core constraint that fights the template, add
`IGNORE_PROJECT_DRUPAL_CORE_VERSION: "1"`.

## 4. Make the jobs that matter actually gate

Upstream ships the lint jobs as `allow_failure: true`, so a red `phpcs` still lets a merge through.
That is the most common reason a project "has CI" and still merges broken code.

```yaml
cspell:
  allow_failure: false
phpcs:
  allow_failure: false
phpstan:
  allow_failure: false
```

Every job you rely on is `allow_failure: false`, and no check is masked with `|| true`. If a lint
config is broken, fix the config — do not silence the job.

## 5. Overriding and extending

Everything after the `include:` is ordinary GitLab CI: redefine a job by name to change its
`variables`, `script`, `rules`, `image` or `allow_failure`, or add your own job in one of the
template's stages. Your definitions must come **after** the include block.

### A custom browser-test job

A theme or site template often needs a real browser suite (webship-js, Playwright, Cucumber, Behat).
The pattern that works on the template's images:

```yaml
.browser-base:
  extends: .testing-job-base
  needs: [composer]
  # The browser comes with the test runner: only the database is needed.
  services:
    - name: $_CONFIG_DOCKERHUB_ROOT/$_TARGET_DB_TYPE-$_TARGET_DB_VERSION:$_TARGET_DB_IMAGE_TAG
      alias: database
  before_script:
    - rm -rf /var/www/html && ln -s $CI_PROJECT_DIR/$_WEB_ROOT /var/www/html
    - service apache2 start
    - $CI_PROJECT_DIR/vendor/bin/drush --root=$CI_PROJECT_DIR/$_WEB_ROOT site:install standard
        --db-url=mysql://$MYSQL_USER:$MYSQL_PASSWORD@database/$MYSQL_DATABASE -y
    - chown -R www-data:www-data $CI_PROJECT_DIR/$_WEB_ROOT/sites/default/files
    # Node and the browser.
    - curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && apt-get install -y nodejs
    - corepack enable && yarn install
    - ./node_modules/.bin/playwright install --with-deps chromium
```

Give the files Drush created back to the web server (`chown`), collect mail instead of sending it
(`system.mail interface.default test_mail_collector`), and keep the reports and failure screenshots
as `artifacts: when: always`.

## 6. Runner time limit and parallel jobs

The shared runners on git.drupalcode.org stop a job after **30 minutes**. A job's own `timeout:`
cannot raise it: a job that declares `timeout: 60m` still ends with

```
ERROR: Job failed: execution took longer than 30m0s seconds
```

No test failed — the job ran out of time. Treat it as a design problem, not a flaky test:

1. **Read the durations** of the last green runs. A job at 25 minutes will cross 30 on a slow runner,
   often on a tag pipeline right after a release.
2. **Remove waste first**: fixed sleeps in step definitions, a forced wait after every page load, a
   full site install repeated inside the suite.
3. **Split the suite with `parallel: N`.** GitLab starts N copies of the job, each with its own
   services, so each gets its own database and site — no shared state between parts. GitLab sets
   `CI_NODE_INDEX` (1…N) and `CI_NODE_TOTAL` (N) in each copy.
4. **Choose the files in the runner's config, not on the command line.** For Cucumber-js, feature
   paths given on the command line do not replace the `paths` of the config file (checked with
   Cucumber-js 12), so every copy would still run the whole suite. Select the files in `cucumber.js` instead:

```js
const fs = require('fs');
const path = require('path');

const featureDir = 'tests/features';
const shardTotal = Number(process.env.CI_NODE_TOTAL || 1);
const shardIndex = Number(process.env.CI_NODE_INDEX || 1);

// A Scenario counts one, and so does each example row of a Scenario Outline.
function scenarioCount(file) {
  let count = 0;
  let rows = -1;
  fs.readFileSync(file, 'utf8')
    .split('\n')
    .forEach((line) => {
      const text = line.trim();
      if (text.startsWith('Scenario:')) {
        count += 1;
        rows = -1;
      } else if (text.startsWith('Examples:')) {
        rows = 0;
      } else if (rows >= 0 && text.startsWith('|')) {
        count += rows > 0 ? 1 : 0;
        rows += 1;
      } else if (text !== '' && !text.startsWith('#')) {
        rows = -1;
      }
    });
  return count;
}

// Largest file first, each to the part with the fewest scenarios so far.
function shardPaths() {
  const shards = Array.from({ length: shardTotal }, () => ({ count: 0, files: [] }));
  fs.readdirSync(featureDir)
    .filter((file) => file.endsWith('.feature'))
    .map((file) => path.join(featureDir, file))
    .map((file) => ({ file, count: scenarioCount(file) }))
    .sort((a, b) => b.count - a.count || a.file.localeCompare(b.file))
    .forEach(({ file, count }) => {
      const lightest = shards.reduce((a, b) => (b.count < a.count ? b : a));
      lightest.files.push(file);
      lightest.count += count;
    });
  return shards[shardIndex - 1].files;
}

module.exports = {
  default: {
    paths: shardTotal > 1 ? shardPaths() : [`${featureDir}/**/*.feature`],
    // …
  },
};
```

```yaml
browser-test:
  extends: .browser-base
  # More scenarios than one job can run in the runner's 30 minutes.
  parallel: 3
  script:
    - ./node_modules/.bin/cucumber-js --config cucumber.js --tags "not @wip"
```

Outside a parallel job the config runs every file, so local runs do not change. Share by scenario
count, not by file count: a plain round-robin can leave one part with nearly twice the work of another.

5. **Prove the split** before pushing: a dry run of every part, whose counts add up to the dry run of
   the whole suite.

   ```bash
   for i in 1 2 3; do
     CI_NODE_TOTAL=3 CI_NODE_INDEX=$i npx cucumber-js --dry-run --format summary
   done
   npx cucumber-js --dry-run --format summary
   ```

6. **Watch for order dependence.** A feature that relied on data another feature created now runs on
   a different site. Fix it in the feature (its own `Given` steps), never by pinning files together.

Pick N for a comfortable margin: each part pays the site install and the browser download again
(a few minutes), so aim for parts well under 20 minutes.

## 7. When the template is not the right answer

A hand-written pipeline is justified when the project genuinely cannot use the template's build —
for example a profile whose release branch deliberately pins a stable dependency, so a full
`composer install` of the profile is not what CI should run. If you write one by hand:

- Declare `stages:` explicitly and keep the names conventional (`build`, `validate`, `test`).
- Add a `workflow:` block so it runs for merge requests, branches and tags:
  ```yaml
  workflow:
    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      - if: $CI_COMMIT_BRANCH
      - if: $CI_COMMIT_TAG
  ```
- Pin images (`php:8.4-cli`, `composer:2`), and set `COMPOSER_ALLOW_SUPERUSER: "1"` +
  `COMPOSER_MEMORY_LIMIT: "-1"`.
- Comment **why** each job exists and why anything is skipped.
- Ship a slim `.gitlab-ci-local.yml` beside it (see `drupal-gitlab-ci-local-runner`).

## 8. Checklist before opening the MR

1. The include block is present and `_GITLAB_TEMPLATES_REF` is **pinned**, not floating.
2. `PROJECT_TYPE` matches reality.
3. Every job you rely on is `allow_failure: false`; nothing is masked with `|| true`.
4. Jobs that cannot apply are skipped **explicitly** with `SKIP_*` and a comment saying why.
5. No job is near the runner limit; long suites are split and the split is proven by a dry run.
6. What can run locally ran green, and the remote pipeline is green.

## References

- Project: <https://www.drupal.org/project/gitlab_templates>
- Docs: <https://project.pages.drupalcode.org/gitlab_templates/>
- Customizations: <https://project.pages.drupalcode.org/gitlab_templates/info/customizations/>
- Variants: <https://project.pages.drupalcode.org/gitlab_templates/info/variants/>
- Test locally: <https://project.pages.drupalcode.org/gitlab_templates/info/test-locally/>
- GitLab `parallel:`: <https://docs.gitlab.com/ci/yaml/#parallel>

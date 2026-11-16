# Chapter 45: CI/CD Provider Deep-Dive

---

A test suite that doesn't run on every commit is a museum piece. The whole point of automated testing is to run automatically, in the background, on a machine that is always paying attention even when humans are not. Webship-js 2.0 ships ready-to-use CI configuration for **thirteen** providers -- GitHub Actions, GitLab CI, Bitbucket Pipelines, Azure Pipelines, CircleCI, Travis, Jenkins, Drone, Semaphore, AWS CodeBuild, Google Cloud Build, Codefresh, and Bamboo. This chapter covers all thirteen, plus six more environments teams keep asking about (TeamCity, Woodpecker, Forgejo, Harness, Octopus, Codeship) where the same pattern ports in minutes.

This chapter is a recipe per provider. Each recipe shows: the moment you reach for it, the config file, the badge, the smallest possible setup, how the pieces fit, and the gotchas. Pick the one you actually use; skim the rest anyway -- the day your company migrates providers, the skim pays for itself.

---

## The Same Pattern, Everywhere

<!-- diagram: cicd-pipeline -->

Every shipped config follows the same shape:

1. Install Node 20 (webship-js requires Node >= 20)
2. Install Playwright + Chromium with system deps
3. Background the app under test (the shipped examples run a fixture server on port 8080 -- the default `launchUrl` of `http://localhost:8080`)
4. Run `npm test`
5. Surface the cucumber-js report as a build artefact

That's it. No build step, no Docker daemon, no custom runners. `npm test` runs cucumber-js with the default profile: step definitions from `tests/step-definitions/**/*.js`, features from `tests/features/**/*.feature`, a 45-second step timeout, and a JSON report written to `tests/reports/cucumber_report.json`. The differences below are syntax, badge URLs, and per-provider quirks.

One thing to internalise before the recipes: everything that varies per environment is an environment variable, never a config edit. `LAUNCH_URL` points the suite at staging or production. `BROWSER` flips chromium/firefox/webkit. `WEBSHIP_VIDEO=on-failure` keeps failure videos. `WEBSHIP_JS_ERROR_MODE=fail` turns console errors into build failures. The YAML stays identical across every project you own -- only the variables change. That one design decision is what lets a single CI template serve forty repositories without forty forks of it.

---

## Recipe 451: GitHub Actions

**Problem:** The suite passes on every laptop on the team, and then a new contributor's first PR breaks checkout in a way nobody notices for two days -- because the tests only run when someone remembers to run them. You want every push and every pull request to trigger the suite on a machine nobody has to babysit, with the report waiting when a human finally looks.

**File:** `.github/workflows/github-actions.yml`. Free for public repos, 2,000 minutes/month for private repos. The single most popular CI/CD by a wide margin.

```yaml
name: Webship-js Tests
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install
      - run: npx playwright install --with-deps chromium
      - run: npm start &
      - run: sleep 3 && npm test
        env:
          FORCE_COLOR: 1
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cucumber-report
          path: tests/reports/
```

**Badge:**
```markdown
[![Tests](https://github.com/USER/REPO/actions/workflows/github-actions.yml/badge.svg)](https://github.com/USER/REPO/actions)
```

**How It Works:** Each `- run:` line gets its own shell, but background processes survive between steps inside the same job -- that is why `npm start &` in one step is still alive when `sleep 3 && npm test` runs in the next. `actions/setup-node@v4` pins Node 20 regardless of what the runner image happens to ship. `FORCE_COLOR: 1` makes cucumber-js emit ANSI colours into the log so failures are scannable. The artifact step uses `if: always()` because the report matters most when the job fails -- artifacts are kept 90 days by default, tunable per step with `retention-days`.

**Gotcha:** Earlier `@v3` action versions emit Node-20 deprecation warnings. Bump to `@v4` to silence them. And remember `ubuntu-latest` is a moving target -- it re-points to a newer OS image every couple of years, and the system libraries move with it. `--with-deps` is what carries Playwright through those migrations without you noticing.

**At scale:** Push every difference into `env:`, not into the YAML. A matrix over `BROWSER: [chromium, firefox, webkit]` triples coverage without a second workflow file, and `LAUNCH_URL` per environment lets the same workflow test staging on PRs and run production smoke checks after deploy. When you own dozens of repos, promote this file to an org-level reusable workflow (`workflow_call`) -- one fix to the template updates forty pipelines, instead of forty near-identical PRs.

---

## Recipe 452: GitLab CI

**Problem:** Your organisation lives on GitLab -- merge requests, issues, the container registry, all of it. The suite runs green on the lead's laptop and red on the shared runner, and now every MR review starts with the same tired question: "is that a real failure or the runner again?" You want the pipeline to be the referee, with the report attached as evidence on every merge request.

**File:** `.gitlab-ci.yml`. 400 compute-minutes/month on shared runners; unlimited on self-hosted. Strong for organisations already on GitLab.

```yaml
image: node:20
stages: [test]
test:
  stage: test
  before_script:
    - npm install
    - npx playwright install --with-deps chromium
  script:
    - npm start &
    - sleep 3
    - npm test
  variables:
    FORCE_COLOR: "1"
  artifacts:
    when: always
    paths: [tests/reports/]
```

**Badge:**
```markdown
[![pipeline](https://gitlab.com/USER/REPO/badges/main/pipeline.svg)](https://gitlab.com/USER/REPO/-/pipelines)
```

**How It Works:** `image: node:20` runs the entire job inside a Node container, so `before_script` installs dependencies and the Chromium binary into that container fresh on every run. The `script` block shares one shell, which is why a plain `npm start &` survives until `npm test` finishes. `artifacts: when: always` uploads `tests/reports/` on success and failure alike -- and on gitlab.com, artifacts expire after 30 days by default, so set `expire_in` explicitly when your audit window is longer than your CI's memory.

**Gotcha:** Shared runners on GitLab.com are slower than GitHub's. For heavy suites, register a runner on your own hardware.

**At scale:** `node:20` is a floating tag -- a routine image rebuild can move one project to a new Node minor overnight while its sibling projects stay behind, and suddenly "identical" suites disagree. Pin the image (`node:20.20` or a digest) and let `package-lock.json` pin Playwright, so every project tests on the same browser build. Then move this whole job into a CI template repository and `include:` it from each project's `.gitlab-ci.yml`: the env vars carry the per-project differences, and a fleet-wide change is one commit in one repo.

---

## Recipe 453: Bitbucket Pipelines

**Problem:** You are an Atlassian shop -- Jira tickets, Bitbucket repos, Confluence specs. The QA lead wants test runs visible next to the pull request, and the free plan's 50 build-minutes are tight enough that every wasted rerun is a line item someone notices. You need the smallest pipeline that runs the suite, caches what it can, and keeps the report.

**File:** `bitbucket-pipelines.yml`. Free for small teams; 50 build-minutes/month on the free tier. Strong for Atlassian shops.

```yaml
image: node:20
pipelines:
  default:
    - step:
        name: Test
        caches: [node]
        script:
          - npm install
          - npx playwright install --with-deps chromium
          - npm start &
          - sleep 3 && npm test
        artifacts: [tests/reports/**]
```

**Badge:**
```markdown
[![Build](https://img.shields.io/bitbucket/pipelines/WORKSPACE/REPO/main)](https://bitbucket.org/WORKSPACE/REPO/pipelines)
```

**How It Works:** `caches: [node]` keys the dependency cache so `npm install` drops from minutes to seconds on warm runs -- on a metered plan, the cache is not a nicety, it is the budget. The whole `script` runs in one container shell, so the backgrounded server survives until the tests finish. `artifacts: [tests/reports/**]` keeps the report downloadable from the Pipelines UI; Bitbucket holds artifacts for 14 days, so anything you must keep longer belongs in external storage (the bucket pattern in Recipe 523 works here too).

**Gotcha:** Bitbucket's pipeline minutes count Linux 1:1 but Windows 4:1. Stick to Linux for cost.

**At scale:** Do the minutes math before you commit the fleet: a 5-minute suite running nightly is roughly 150 minutes a month *per project* -- the free tier covers one project running weekly, not ten running nightly. Teams that stay on Bitbucket buy minutes or attach a self-hosted runner; teams that don't usually keep the repos and move CI elsewhere. Either way the webship-js side never changes -- only the YAML wrapper does, which is exactly the point of this chapter.

---

## Recipe 454: CircleCI

**Problem:** Your suite takes long enough on slow shared runners that the team has stopped waiting for it -- they merge first and check the result later, which quietly defeats the entire point of CI. CircleCI's fast container start and generous free tier are the draw; a config that doesn't fight its execution model is the price of admission.

**File:** `.circleci/config.yml`. 6,000 build-minutes/month free. The shipped config uses `cimg/node:20.20` -- the lightweight image dropped `apt upgrade` and unused tools for faster start.

```yaml
version: 2.1
jobs:
  test:
    docker:
      - image: cimg/node:20.20
    steps:
      - checkout
      - run: npm install
      - run: npx playwright install --with-deps chromium
      - run:
          command: npm start
          background: true
      - run: sleep 3 && npm test
      - store_artifacts:
          path: tests/reports
workflows:
  build:
    jobs: [test]
```

**Badge:**
```markdown
[![CircleCI](https://circleci.com/gh/USER/REPO.svg?style=shield)](https://circleci.com/gh/USER/REPO)
```

**How It Works:** CircleCI runs every `run` step in its own shell and does not let stray processes drift across steps -- that is why the fixture server uses CircleCI's native `background: true` instead of a bare `&`, which would die with its step's shell. The `cimg/node:20.20` image is pinned to an exact Node minor, which is the quiet hero of this config: the runtime cannot drift under you between Tuesday and Thursday. `store_artifacts` exposes `tests/reports` in the job UI, retained for 30 days.

**Gotcha:** The `cimg/node` image includes browsers system deps -- no `--with-deps` needed. The shipped config still passes the flag for safety on custom images.

**At scale:** That pinned image tag is the policy worth copying to every provider you run: pin the image, pin Playwright (1.58 in webship-js 2.0), and upgrade both deliberately, on your schedule, across all projects in one coordinated wave. Unpinned images are how "nothing changed but three suites started flaking" arrives at 2 a.m. -- a fresh browser build alters a focus or scroll behaviour, and the diff you are hunting for does not exist in any of your repos.

---

## Recipe 455: Travis CI

**Problem:** You inherit a repository with a `.travis.yml` from 2019 and a README badge stuck on "build: unknown". The team asks whether to revive it or migrate, and nobody wants to spend a sprint finding out. This recipe is the honest answer: here is the working config, and here is why you probably shouldn't invest in it.

**File:** `.travis.yml`. The shipped config replaces the long-defunct `.travis-ci.yml` (`org` is dead; `com` is the survivor).

```yaml
language: node_js
node_js: [20]
addons:
  chrome: stable
script:
  - npm install
  - npx playwright install --with-deps chromium
  - npm start &
  - sleep 3 && npm test
```

**Badge:**
```markdown
[![Build Status](https://app.travis-ci.com/USER/REPO.svg?branch=main)](https://app.travis-ci.com/USER/REPO)
```

**How It Works:** `language: node_js` provisions the Node 20 toolchain before your scripts run. The `addons: chrome: stable` line predates Playwright bundling its own browsers and is harmless but redundant -- `npx playwright install --with-deps chromium` is what actually provides the browser the tests drive. All `script` lines share one shell, so the standard background-server pattern works unchanged.

**Gotcha:** Travis's free tier was discontinued for open source in 2021. Most projects migrate off. The shipped config exists for legacy reasons; new projects should pick GitHub Actions. If you maintain a portfolio, treat every `.travis.yml` you find as a migration ticket waiting to be filed, not a system to extend.

---

## Recipe 456: Jenkins

**Problem:** Your company runs Jenkins on its own iron -- compliance wants builds inside the network, and nobody is migrating two hundred jobs to a SaaS this quarter. You get unlimited minutes and total control; in exchange you own Node installs, agent hygiene, and every plugin upgrade. You want the smallest Jenkinsfile that runs the suite and archives the evidence.

**File:** `Jenkinsfile`. Self-hosted -- no usage limits, but you own the runners.

```groovy
pipeline {
  agent any
  tools { nodejs '20' }
  stages {
    stage('Install') { steps { sh 'npm install' } }
    stage('Browsers') { steps { sh 'npx playwright install --with-deps chromium' } }
    stage('Test') {
      steps {
        sh 'npm start &'
        sh 'sleep 3 && npm test'
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'tests/reports/**', allowEmptyArchive: true
    }
  }
}
```

**Badge:** Jenkins doesn't expose a public badge by default. Install the Jenkins Build Status plugin and point a shields.io endpoint at it.

**How It Works:** Each `sh` step runs in its own shell on the agent. The declarative `post { always { archiveArtifacts ... } }` block is Jenkins' answer to `if: always()` -- the report is attached to the build page whether the suite passed or not, and retention follows the job's build-discarder policy, which (unlike SaaS providers) you control completely. `tools { nodejs '20' }` resolves Node from a Global Tool Configuration entry at runtime, so agents don't need Node baked into their images.

**Gotcha:** Jenkins agents need Node 20 installed (via NodeJS plugin). The `tools { nodejs '20' }` block references a Global Tool configuration -- set it up in `Manage Jenkins → Global Tool Configuration`. One more Jenkins-specific trap: a bare `sh 'npm start &'` can leave the step waiting on the open log pipe, or get the server reaped by Jenkins' process cleanup. If the Test stage hangs or hits connection-refused, launch the server with `nohup`, redirect its output, and set `JENKINS_NODE_COOKIE=dontKillMe` on it.

**At scale:** Forty projects means forty Jenkinsfiles drifting apart unless you stop it early. Put this pipeline in a Jenkins Shared Library, reduce each repo to a one-line `webshipPipeline()` call, and pass the per-project differences -- `LAUNCH_URL`, the tag expression to run -- as parameters. Agent hygiene is the other half: label the agents that have browser system libraries and pin webship jobs to that label, or `--with-deps` will fail on the one agent in the pool that's missing a package, and it will always be the 3 a.m. run that lands there.

---

## Recipe 457: Azure Pipelines

**Problem:** Your .NET services already build in Azure DevOps -- boards, repos, releases, the lot -- and the release manager keeps asking why the UI tests live somewhere else. Every release review means stitching evidence from two systems, and the one time it mattered, the link to the external test run had expired. You want the webship-js suite in the same pipeline view as the deployment it guards.

**File:** `azure-pipelines.yml`. 1,800 free minutes/month for public projects; private projects need a paid plan after the free hosted parallel job expires.

```yaml
trigger: [main]
pool:
  vmImage: 'ubuntu-latest'
steps:
- task: NodeTool@0
  inputs: { versionSpec: '20.x' }
- script: npm install
- script: npx playwright install --with-deps chromium
- script: npm start &
- script: sleep 3 && npm test
  env: { FORCE_COLOR: 1 }
- task: PublishPipelineArtifact@1
  condition: always()
  inputs:
    targetPath: 'tests/reports'
    artifactName: 'cucumber-report'
```

**Badge:**
```markdown
[![Build](https://dev.azure.com/ORG/PROJECT/_apis/build/status/REPO)](https://dev.azure.com/ORG/PROJECT/_build)
```

**How It Works:** `NodeTool@0` resolves Node 20.x at runtime, so the suite is not hostage to whatever the `ubuntu-latest` image happens to carry this quarter. Each `- script:` step gets its own shell, but they all run on the same hosted agent, so the backgrounded fixture server from one step is still listening when the next step runs the tests -- the same model as GitHub Actions. `PublishPipelineArtifact@1` with `condition: always()` is the piece teams forget: without it, a red run uploads nothing, and a red run is precisely when you need `tests/reports/` the most.

**Gotcha:** First-time setup requires creating an Azure DevOps organisation. The free parallel job request takes 2-3 business days to be approved by Microsoft.

**At scale:** the unit Microsoft meters is the *parallel job*, granted per organisation, not per project. One free hosted job means your forty pipelines run one at a time -- a nightly fleet that should finish by 2 a.m. is still queuing at breakfast. Either buy parallel jobs or register self-hosted agents (and if you self-host, bake the Playwright system deps into the agent image so `--with-deps` stops being a per-run download). Put `LAUNCH_URL` and friends in a variable group in the Azure DevOps Library: one shared set of environment wiring, linked into every pipeline, instead of forty copies drifting apart.

---

## Recipe 458: AWS CodeBuild

**Problem:** Security finally signs off on automated testing -- with one condition: the staging environment stays inside the VPC, unreachable from any SaaS runner on the public internet. Your infrastructure is already Terraform-on-AWS, the team already lives in CloudWatch, and standing up a separate CI vendor for one test suite is a procurement conversation nobody wants. CodeBuild runs inside your account, inside your network, on your IAM roles.

**File:** `buildspec.yml`. Pay-per-minute. The lower compute classes are the cheapest; `BUILD_GENERAL1_SMALL` is plenty for webship-js.

```yaml
version: 0.2
phases:
  install:
    runtime-versions: { nodejs: 20 }
    commands:
      - npm install
      - npx playwright install --with-deps chromium
  build:
    commands:
      - npm start &
      - sleep 3 && npm test
artifacts:
  files: [tests/reports/**/*]
```

**Badge:** AWS CodeBuild does not ship native badges. Wire a CloudWatch alarm to an SNS topic and surface via a shields.io endpoint or a custom dashboard.

**How It Works:** `runtime-versions: nodejs: 20` in the install phase selects the managed Node runtime -- no setup task, no curl-pipe-bash. Each command in a buildspec gets a fresh shell, but every command shares the same build container, so the backgrounded `npm start` keeps serving while `sleep 3 && npm test` runs as the next command. The `artifacts` block hands `tests/reports/**/*` to the S3 bucket configured on the CodeBuild project -- which means report retention is not a CI setting here at all, it is whatever lifecycle policy you put on that bucket. For VPC-internal staging, attach the project to the VPC and point `LAUNCH_URL` at the internal hostname; the suite neither knows nor cares that no public route exists.

**Gotcha:** CodeBuild has no built-in scheduler. Trigger from EventBridge (cron) or from a webhook on your source provider (CodeCommit, GitHub, Bitbucket).

**At scale:** pay-per-minute makes the fleet cost an arithmetic problem instead of a plan-tier surprise: forty suites at five minutes nightly on `BUILD_GENERAL1_SMALL` is a predictable, boring number -- and boring is what you want when finance asks. Keep it boring by splitting runs the way the tag chapter teaches: `@critical` on every push, the full suite on the EventBridge nightly. And because artifacts land in S3 anyway, this is the one provider where long-term evidence storage needs no extra step -- one lifecycle rule (transition to infrequent access at 90 days) covers every project in the account.

---

## Recipe 459: Google Cloud Build

**Problem:** The application deploys to Cloud Run, secrets live in Secret Manager, and the team authenticates with workload identity instead of exported keys -- then the test suite shows up needing a service-account JSON file pasted into a third-party CI vendor, and the platform engineer vetoes it on the spot. Keeping the suite inside the same GCP project ends the credential-export argument before it starts.

**File:** `cloudbuild.yaml`. 120 free build-minutes per day. Pay-per-minute beyond that. Tight integration with the rest of Google Cloud.

```yaml
steps:
- name: 'node:20'
  entrypoint: 'bash'
  args:
    - '-c'
    - |
      npm install
      npx playwright install --with-deps chromium
      npm start &
      sleep 3
      npm test
artifacts:
  objects:
    location: 'gs://YOUR_BUCKET/cucumber-reports/'
    paths: ['tests/reports/**']
```

**Badge:** Not native. Use a shields.io endpoint that queries Cloud Build's REST API.

**How It Works:** The whole job is one step running the `node:20` container with a `bash` entrypoint, and the `|` block hands bash a single multi-line script -- one shell, so `npm start &` survives until `npm test` exits, no provider-specific background mechanism needed. The `artifacts.objects` block copies `tests/reports/**` straight into a GCS bucket, which (like CodeBuild's S3 pattern) moves retention out of the CI tool and into a bucket lifecycle rule you control. Pin the image to a digest rather than the floating `node:20` tag when you need build-to-build reproducibility.

**Gotcha:** Cloud Build's default machine type is small. For Playwright-heavy suites, set `options.machineType: 'E2_HIGHCPU_8'`.

**At scale:** the 120 free daily minutes are per billing project, and a ten-suite nightly fleet eats them before midnight -- after that you are paying per minute, so the machine-type gotcha becomes a cost dial: `E2_HIGHCPU_8` costs more per minute but finishes Playwright work so much faster that the bill often comes out flat. Use substitution variables (`_LAUNCH_URL`, `_BROWSER`) in one shared `cloudbuild.yaml` and set them per trigger; every product gets its own staging URL and browser without forking the file, which is the same env-var-not-config-edit rule this chapter opened with.

---

## Recipe 460: TeamCity

**Problem:** The company has run TeamCity on-prem for a decade -- every Java service builds there, the release dashboard lives there, and ops knows its failure modes by heart. Then the web team adopts webship-js, and suddenly the QA suite is the one build that lives somewhere else: invisible on the dashboard, absent from the release gate. Adding one more vendor for one suite is a harder sell than teaching TeamCity to run Node.

**File:** `.teamcity/settings.kts` (plus `pom.xml`). Kotlin DSL. JetBrains' on-prem CI -- common in JVM shops.

```kotlin
import jetbrains.buildServer.configs.kotlin.v2019_2.*

version = "2024.03"
project {
  buildType(BuildType {
    id("Webship_Tests")
    name = "Webship-js Tests"
    steps {
      script {
        name = "Install + Test"
        scriptContent = """
          npm install
          npx playwright install --with-deps chromium
          npm start &
          sleep 3
          npm test
        """.trimIndent()
      }
    }
    artifactRules = "tests/reports => cucumber-report"
  })
}
```

**Badge:** TeamCity exposes a status URL but no SVG. Wrap via shields.io endpoint.

**How It Works:** The Kotlin DSL is configuration-as-code in the literal sense -- the build definition lives in the repo, so a change to how tests run is a reviewable diff instead of a click someone made in the admin UI last spring. `scriptContent` is one script in one shell, which is why the plain `npm start &` background pattern works unmodified. `artifactRules = "tests/reports => cucumber-report"` publishes the report directory under a stable artifact name; retention follows the server's clean-up rules, which on-prem means *your* rules. The build agent must have Node 20 available -- TeamCity will not provision it for you the way `setup-node` does.

**Gotcha:** The Kotlin DSL needs the TeamCity server to be on a recent version (2019.2+). Older on-prem instances may need the XML format instead.

---

## Recipe 461: Semaphore

**Problem:** Four in the afternoon, six PRs queued, and the team has developed the habit that quietly kills CI cultures: merge now, check the pipeline later, because waiting for it costs more attention than ignoring it. The suite is not slow -- the feedback loop is. Semaphore's pitch is raw start-up speed: machines boot in seconds, so the time between push and verdict shrinks to something a developer will actually sit through.

**File:** `.semaphore/semaphore.yml`. 1,300 free minutes/month. Fast Docker-based provider with a clean UI.

```yaml
version: v1.0
name: Webship-js
agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu2204
blocks:
- name: Test
  task:
    jobs:
    - name: cucumber
      commands:
      - checkout
      - npm install
      - npx playwright install --with-deps chromium
      - nohup npm start &
      - sleep 3
      - npm test
    epilogue:
      always:
        commands:
          - artifact push job tests/reports
```

**Badge:**
```markdown
[![Build](https://USER.semaphoreci.com/badges/REPO/branches/main.svg)](https://USER.semaphoreci.com/projects/REPO)
```

**How It Works:** The `agent` block picks the VM size (`e1-standard-2`) and OS image (`ubuntu2204`) explicitly -- there is no `ubuntu-latest` ambiguity to drift under you. `checkout` is Semaphore's built-in clone command, the job's commands share a session, and the `epilogue.always` block plays the same role as `if: always()` elsewhere: `artifact push job tests/reports` uploads the report whether the suite passed or failed. Job artifacts are the short-lived tier; promote anything you must keep to project-level artifacts or external storage.

**Gotcha:** Semaphore uses a long-lived `nohup` for background processes -- `npm start &` alone can be reaped.

---

## Recipe 462: Drone CI

**Problem:** The compliance ruling is blunt: source code and build logs never leave the network. Jenkins would do it, but nobody on the team wants to inherit a plugin garden for one Node test suite -- you want something you can run as a single container, configure with one YAML file, and explain to the auditor in a sentence. That is Drone's whole personality.

**File:** `.drone.yml`. Open-source, self-hosted. Increasingly used as a lightweight alternative to GitLab CI.

```yaml
kind: pipeline
type: docker
name: webship-js
steps:
- name: test
  image: node:20
  commands:
  - npm install
  - npx playwright install --with-deps chromium
  - npm start &
  - sleep 3 && npm test
```

**Badge:** Use a shields.io endpoint pointed at the Drone server's status API.

**How It Works:** `type: docker` means every step is a container; the `test` step pulls `node:20`, runs all four commands in one shell inside it, and throws the container away. That is the cleanest isolation model in this chapter -- nothing persists between runs except what you cache deliberately -- and it is also why the background-server pattern works: server and tests share one container, one shell, one lifetime.

**Gotcha:** Drone has been quiet on releases since the Harness acquisition. Many users have moved to Woodpecker (a Drone fork; see next recipe). The same `.drone.yml` works in both.

**At scale:** self-hosting puts you on the other side of the image-supply problem. Forty repos each pulling `node:20` from Docker Hub on every run will meet the rate limiter at the worst possible hour -- mirror the image in an internal registry and pin it by digest, so every project tests on the byte-identical runtime and the 2 a.m. run cannot be broken by an upstream rebuild. The portability is the other payoff: the same `.drone.yml` runs under Woodpecker verbatim, so the migration path off a stagnating server is a DNS change, not a rewrite.

---

## Recipe 463: Woodpecker CI

**Problem:** The annual security review flags your Drone server: release cadence near zero since the Harness acquisition, CVE response unclear, and the auditor wants a maintenance story you cannot give. Migrating CI usually means a quarter of rewrites -- except here, where the community fork was built to accept your existing config unchanged.

**File:** reuses `.drone.yml`. A community fork of Drone, MIT-licensed and actively maintained.

```yaml
# Same as Drone's .drone.yml -- Woodpecker accepts it verbatim
when:
  branch: [main]
steps:
  test:
    image: node:20
    commands:
      - npm install
      - npx playwright install --with-deps chromium
      - npm start &
      - sleep 3 && npm test
```

**Badge:**
```markdown
[![status](https://WOODPECKER_SERVER/api/badges/USER/REPO/status.svg)](https://WOODPECKER_SERVER)
```

**How It Works:** Same execution model as Drone -- each step is a disposable container, commands share one shell inside it -- with a `when: branch` filter shown here so the suite runs on `main` pushes rather than every branch. The webship-js side is untouched by the fork: `LAUNCH_URL`, `BROWSER`, and the rest of the env-var surface behave identically, because the suite never knew which server was running it in the first place.

**Gotcha:** Drone's "kind/type" block is optional in Woodpecker but still parses correctly. New Woodpecker projects can use the leaner native syntax.

---

## Recipe 464: Forgejo Actions

**Problem:** The organisation moved its repos to a self-hosted Forgejo instance -- data sovereignty, no US cloud dependency, the usual reasons -- and the migration script carried everything across except one thing that matters daily: the `.github/workflows/` directory that used to run the tests. Rewriting forty working workflows for a new CI dialect is exactly the tax the migration plan promised to avoid.

**File:** reuses `.github/workflows/*`. Forgejo (the Codeberg-driven Gitea fork) has a GitHub-Actions-compatible runner.

```yaml
# Identical syntax to GitHub Actions; runs unmodified.
name: Webship-js Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: docker
    container:
      image: node:20
    steps:
      - uses: https://code.forgejo.org/actions/checkout@v4
      - run: npm install && npx playwright install --with-deps chromium
      - run: npm start &
      - run: sleep 3 && npm test
```

**Badge:**
```markdown
[![status](https://FORGEJO_HOST/USER/REPO/actions/workflows/main.yml/badge.svg)](https://FORGEJO_HOST/USER/REPO/actions)
```

**How It Works:** The Forgejo runner speaks the GitHub Actions protocol, so the workflow syntax -- jobs, steps, `uses:`, `run:` -- carries over unmodified. Two deliberate differences in the file above: `runs-on: docker` with an explicit `container.image: node:20` replaces GitHub's hosted `ubuntu-latest` (you own the runner, so you own the image), and the checkout action is pinned to its full `code.forgejo.org` URL so the runner never silently reaches out to a registry you did not choose. Everything below the YAML -- the install, the backgrounded server, `npm test`, the report -- is byte-for-byte the GitHub Actions recipe.

**Gotcha:** Forgejo's action runner uses the same protocol as GitHub but pulls action images from `code.forgejo.org` by default. Pin actions explicitly if you mix runners.

---

## Recipe 465: Harness CI

**Problem:** Leadership consolidates: one platform for CI, CD, and feature flags, signed at the executive level, and every team's pipelines are expected to move in by Q3. You did not pick Harness -- it picked you. The question is not "is this the right CI" but "what is the smallest pipeline definition that gets the suite running inside it without disturbing the suite itself".

**File:** `.harness/webship-js-pipeline.yml`. Enterprise CI/CD with strong feature-flag and deployment tooling baked in.

```yaml
pipeline:
  name: Webship-js Tests
  identifier: webship_js
  stages:
  - stage:
      name: Test
      type: CI
      spec:
        execution:
          steps:
          - step:
              type: Run
              name: Test
              spec:
                connectorRef: docker-hub
                image: node:20
                command: |-
                  npm install
                  npx playwright install --with-deps chromium
                  npm start &
                  sleep 3 && npm test
```

**Badge:** Harness offers status badges via its API; configure a public dashboard or embed via shields.io.

**How It Works:** The nesting is deep -- pipeline, stage, execution, step -- but the working core is one `Run` step: pull `node:20` through the `connectorRef` (Harness's named credential for Docker Hub), run the familiar four commands in one shell. The `command: |-` block is the same single-shell pattern as Cloud Build, so the backgrounded server needs no special handling. The execution itself happens on a *delegate* -- an agent you run in your own cluster or VM -- which is why the suite can reach internal staging URLs the SaaS control plane never sees.

**Gotcha:** Harness pipelines bind tightly to its delegate model -- you'll need a delegate running in the cluster or VM that executes the build.

---

## Recipe 466: Bamboo Data Center

**Problem:** The Bamboo Data Center renewal just went through for three more years, the build agents sit in the company server room, and the change-advisory board is not entertaining a CI migration this cycle. The web team still needs its webship-js suite running per push. The pragmatic move is to meet the estate where it is: a Bamboo Specs file in the repo, the same suite underneath.

**File:** `bamboo-specs/bamboo.yml`. Atlassian's on-prem CI for enterprise shops still on the Atlassian stack.

```yaml
version: 2
plan:
  project-key: WS
  key: TEST
  name: Webship-js Tests
stages:
- Test:
  - jobs:
    - Test:
        tasks:
        - script:
            scripts:
            - |
              npm install
              npx playwright install --with-deps chromium
              npm start &
              sleep 3 && npm test
        artifacts:
          - name: cucumber-report
            location: tests/reports
            pattern: '**'
```

**Badge:** Bamboo exposes a JSON status endpoint -- wire to shields.io.

**How It Works:** Bamboo Specs is Bamboo's configuration-as-code: the YAML maps to a project (`WS`), a plan (`TEST`), and a script task whose multi-line block runs in one shell -- background-server pattern intact. The `artifacts` entry publishes `tests/reports` with pattern `'**'` so the whole report directory (HTML, JSON, screenshots) attaches to each build result; retention follows the plan's expiry settings, which on Data Center are yours to set as generously as the audit window demands. As with TeamCity and Jenkins, the agent must already have Node 20 -- provision it in the agent image, not in the plan.

**Gotcha:** Atlassian deprecated Bamboo Cloud years ago. Only the Data Center / Server editions remain. Any new project should consider Bitbucket Pipelines instead, which is the modern Atlassian CI offering.

---

## Recipe 467: Codefresh

**Problem:** Everything else in the platform is GitOps already -- deployments reconcile through Argo, environments are declared, drift is a dirty word -- and the test suite is the last piece still running on a VM-shaped CI that thinks in shell scripts. The platform team wants CI that speaks containers natively, so the test pipeline composes like the rest of the stack.

**File:** `codefresh.yml`. Container-native CI/CD with first-class GitOps support.

```yaml
version: '1.0'
stages: [prepare, test]
steps:
  install:
    title: Install dependencies
    image: node:20
    stage: prepare
    commands:
      - npm install
      - npx playwright install --with-deps chromium
  test:
    title: Run tests
    image: node:20
    stage: test
    commands:
      - npm start &
      - sleep 3 && npm test
```

**Badge:** Native badge support via project settings.

**How It Works:** Every step is a container, but -- unlike Drone, where each step's container is disposable and isolated -- Codefresh mounts one shared volume across all steps in the pipeline. That is the trick that makes this two-step layout work: the `install` step's `node_modules` and browser download are still on disk when the `test` step starts in a fresh `node:20` container. Within the `test` step, both commands share a shell, so the backgrounded fixture server lives exactly as long as the tests need it.

**Gotcha:** Codefresh's free tier removed unlimited builds in 2023 -- check current pricing for non-OSS use.

---

## Recipe 468: Octopus Deploy (CD-Only)

**Problem:** The release manager runs deployments through Octopus and asks the reasonable-sounding question: "can Octopus run the tests too, so everything lives in one tool?" Someone has to give the honest answer before a sprint disappears into making a deployment tool impersonate a CI server.

**File:** no test runner -- Octopus is deployment-only. Webship-js tests are invoked from upstream CI (GitHub Actions, Jenkins, etc.) and Octopus deploys the validated artefact.

**Pattern:**

```
GitHub Actions  →  npm test  →  upload artefact  →  Octopus Deploy
                                                    (downloads artefact + ships)
```

**Why list it here:** Many teams ask "can Octopus run my tests?". Strictly no -- Octopus runs deployments, not test suites. Pair it with a CI provider that runs webship-js, then deploy the green build through Octopus. The division of labour is actually a feature: CI produces the green run *and* the `tests/reports/` evidence, the artefact Octopus deploys is the one that evidence describes, and the release page can link straight to the report that justified shipping it. A post-deploy smoke run -- the same suite with `LAUNCH_URL` pointed at production and a `@smoke` tag filter -- still belongs to the CI side, triggered by Octopus's deploy hook.

---

## Recipe 469: Codeship (Retired)

**Problem:** A repo you inherit still carries `codeship-steps.yml`, a badge that resolves to nothing, and a README that swears the tests run on every push. They have not run since 2023.

**File:** none -- Codeship was retired in 2023.

**Action:** If you have a legacy `codeship-steps.yml` or `codeship-services.yml`, migrate to CloudBees CI (the post-acquisition successor) or pick any of the providers above. The translation is mechanical -- a Codeship config maps straight onto the five-step shape from the top of this chapter -- and deleting the dead config matters as much as adding the new one, because a badge that lies is worse than no badge at all.

---

## Recipe 470: Choosing a Provider

The question rarely arrives as "which CI is best". It arrives as a merger, a compliance ruling, a pricing change, or a new platform lead with opinions -- and suddenly someone has to pick, this week, for every project at once. The good news: because the suite's entire per-environment surface is environment variables, the choice is lower-stakes than it feels. You are picking a YAML wrapper, not re-architecting your tests.

The decision tree, simplified:

```
Public repo?               → GitHub Actions (free, unlimited)
GitLab shop?               → GitLab CI (free shared runners, unlimited self-hosted)
Atlassian shop?            → Bitbucket Pipelines (smaller suites) or Bamboo Data Center
Self-host required?        → Jenkins, Drone, Woodpecker, or Forgejo
On Azure?                  → Azure Pipelines
On AWS?                    → CodeBuild (paired with CodePipeline)
On GCP?                    → Cloud Build
JVM shop, on-prem JetBrains? → TeamCity
Container-native + GitOps? → Codefresh or Harness
Just want fast results?    → Semaphore or CircleCI
```

The shipped configs let you pivot. Drop a new file, push, done. No re-implementation needed.

**At scale:** pick *one* provider for the portfolio and defend that choice harder than the choice itself. Twenty projects on one provider means one template, one secrets convention, one place to pin browser images, and one bill to explain; twenty projects on four providers means four of everything and a platform engineer who spends Fridays translating YAML dialects. When an acquisition hands you repos on a second provider, run both only as long as the migration takes -- and let the env-var rule do the moving: if a suite needs anything beyond `LAUNCH_URL`, `BROWSER`, and friends to run somewhere new, that is a defect in the suite, not in the provider.

---

## CI/CD Gotchas Common to All Providers

**1. Background server reaping:** Some providers kill background processes between steps. Use `nohup`, `&`, or `disown` per the provider's docs.

**2. Sleep before test:** Always give the fixture server time to bind: `sleep 3 && npm test`. Without it, the first test hits a refused connection and fails for the wrong reason.

**3. Browser system deps:** `npx playwright install --with-deps chromium` installs both the browser binary and the system libraries it needs. Skip `--with-deps` only on images you know already have them (cimg/node, mcr.microsoft.com/playwright).

**4. Color output:** Set `FORCE_COLOR=1` in the job environment so cucumber-js emits ANSI colours. Without it, the report is hard to read.

**5. Artefacts always:** Upload reports on success and failure -- otherwise you lose the failure evidence when you need it most. Most provider configs accept `always()` or `if: always:` for this.

**6. Cache `node_modules`:** Most providers key the dependency cache on a hash of `package-lock.json`. Adds ~30s the first run; saves a minute every run after -- multiplied by every run of every project, which is where metered plans live or die.

The shipped configs already handle all six. If you adapt them for a provider not in this chapter, port the same six concerns -- you'll skip an hour of debugging.

---

## A Note on Reports

Every provider config above uploads `tests/reports/` as a build artefact. Inside that directory you'll find:

- `cucumber_report.html` -- human-readable report with embedded screenshots
- `cucumber_report.pdf` -- print-friendly version
- `cucumber_report.json` -- machine-readable (parse for dashboards)
- `screenshots/` -- failure screenshots
- `videos/` -- (if recording enabled)

Point your post-CI tools at the JSON file. Push the HTML to S3, GCS, or a static-hosting service for browser-friendly access. The report is the most valuable artefact your suite produces -- preserve it.

One more habit that pays off across a portfolio: stamp the report with run metadata. `bin/generate-reports.js` takes `--app-version`, `--env`, `--browser`, `--platform`, and arbitrary `--metadata Key=Value` pairs -- wire your CI variables into those flags and every report says exactly which build, environment, and browser produced it. That is what turns forty JSON files into one comparable dashboard, and it is the difference between "the suite failed" and "release 2.4.1 failed on staging, Firefox only" when a manager asks.

---

CI/CD is the bridge between writing tests and trusting them. With thirteen ready-to-use configs shipped in the repo and six more environments mapped in this chapter, the bridge is already built. Pick the one that matches the runners you have, push, and watch the green.

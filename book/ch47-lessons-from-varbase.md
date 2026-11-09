# Chapter 47: Lessons From Varbase

---

A lot of webship-js's design did not come from a whiteboard. It came from years of running automated acceptance tests against **Varbase**, the enterprise Drupal distribution maintained at Vardot. Varbase is a starter kit used by global non-profits, governments, universities, and media houses. Every release ships through a Cucumber-driven acceptance suite that has, over time, grown into one of the most-tested Drupal distributions in the world. The lessons in this chapter come from that suite -- patterns we kept, patterns we ditched, and surprises that changed how we wrote later steps.

If you want to read the source, the test fixtures live at:

- `github.com/Vardot/varbase/tree/10.1.x/tests/features/varbase`
- `github.com/Vardot/varbase-project/tree/11.0.x/tests/features`
- `github.com/Vardot/varbase-project/blob/11.0.x/tests/step-definitions/varbase-step-definitions.js`

Reading them is the fastest tour of webship-js in production. Below are the lessons distilled into recipes.

### Three Principles From the Practice

Much of how the Varbase suite is written traces back to a couple of rules of thumb Mohammed Razem pressed on the team early, and which the suite then proved out release after release:

> **Test the scenario, not the configuration.** Assert what a user can do, not what a settings form contains, and do not walk the configuration checkbox by checkbox. Configuration drifts and gets refactored; the behaviour it is supposed to produce is the contract worth pinning. A scenario that logs in and creates an article survives a dozen config refactors that a "the checkbox is checked" assertion would break on.

> **One basic scenario exercises the whole stack.** A single plain path, log in, create, save, see it published, runs through the front-end, the back-end, *and* the configuration that wires them together, all in one pass. So a small number of basic scenarios covers what dozens of config-by-config checks would, and catches the breakages that actually ship, without an elaborate suite. Varbase was built on exactly this, which is why its suite stays fast enough to run on every release.

These two show up throughout the recipes below.

---

## How the Varbase Test Suite Is Organised

<!-- diagram: webship-ecosystem -->

Early Varbase shipped five numbered groups. A fresh Varbase 11 install has grown that into a **fifteen-group matrix**, still in the order a real installation builds itself up:

```
01-website-base-requirements   ─ smoke tests, install profile, base config
02-user-management             ─ login, password reset, role assignment
03-admin-pages                 ─ admin surfaces, reports, status
04-admin-users                 ─ user CRUD, masquerade, account flows
05-content-pages               ─ landing and utility pages
06-content-blog                ─ blog listing, post creation
07-content-contact             ─ contact forms, submissions
08-content-homepage            ─ homepage components and layout
09-drupal-canvas               ─ the Canvas visual editor
10-content-permissions         ─ per-role content access matrix
11-content-workflow            ─ moderation states, transitions
12-content-access-and-lock     ─ unpublished access, content lock
13-varbase-recipes             ─ recipe-driven feature installs
14-ai                          ─ AI agents and assisted authoring
15-quality                     ─ accessibility, performance, audits
```

Every scenario carries environment tags: `@local @development @staging @production`. The tag set determines where the scenario runs. The smoke tests run everywhere; the destructive admin tests run only on `@local` and `@development`.

The matrix grew because the product did, AI authoring, the Canvas editor, recipe installs, and a dedicated quality group are all newer than the original five, but the shape did not change: numbered groups, environment-aware tags, smoke-first ordering. That is the shape recommended in Chapter 19, and it comes from years of pain on Varbase. Other organisations were tried: by module, by author, one giant directory. Each collapsed under its own weight within a year. The numbered-group shape survived because it encodes two things at once, *dependency order* (groups run in the order a real installation builds itself up) and *blast radius* (a failure in `01-` means stop reading, everything downstream is noise). Adding a group is just appending the next number, so the suite scales without ever being reorganised, and anyone who has worked on one Varbase suite can navigate all fifteen groups without a map.

---

## Recipe 526: Smoke First, Then Depth

### Lesson
It is 11pm on release night and the full suite has been running for forty minutes when scenario 214 fails -- because the install profile broke an hour ago and nothing was watching. Every failure after that is noise from the same root cause. Run the cheapest, broadest tests first. If the foundation broke, you want the first failure to be the actionable one, three minutes in, not the two-hundredth, forty minutes in.

### Pattern from Varbase
The `01-website-base-requirements/` directory contains tests like "homepage loads", "admin login page renders", "system status report has zero errors". They run in 30 seconds. If they fail, the rest of the suite is skipped.

```gherkin
@smoke @local @development @staging @production
Feature: Website base requirements

  Scenario: Homepage is reachable
    Given I am on "/"
    Then I should see "Welcome"
    And there should be no JavaScript errors

  Scenario: System status is clean
    Given I am a logged in user with the "Webmaster" user
    And I am on "/admin/reports/status"
    Then I should not see "Errors"
```

### Recipe

Reserve the lowest-numbered group for tests that take seconds and cover assumptions every other test depends on. If those fail, abort early. Wire it via Cucumber's `--tags @smoke` flag on a fail-fast CI run before the full suite.

There is more happening in those two scenarios than the Gherkin shows. Every `Given I am on` navigation goes through webship-js's smartSettle -- fetch/XHR counters at zero, no pending timers, DOM quiet for 250ms -- so "homepage is reachable" really means "homepage loaded, finished its AJAX, and stopped mutating." And because webship-js watches the console on every step (`WEBSHIP_JS_ERROR_MODE` defaults to `warn`; flip it to `fail` for smoke runs, or tag the feature `@js-fail`), the smoke pass doubles as a JavaScript health check without a single extra step.

**At scale:** the smoke gate is where this pattern pays for itself. At Vardot the same `@smoke` convention runs across every project pipeline, so one CI template serves all of them: stage one runs `--tags @smoke` and kills the build in under five minutes if red; stage two runs the full suite only on green. Across dozens of nightly pipelines, that single gate is the difference between burning forty runner-minutes per broken site and burning four.

---

## Recipe 527: Environment-Aware Tags

### Lesson
A test that drops the database is fine on `@local`. The same test on `@production` would be a career-limiting move -- and the scary part is how easy that move is: one wrong `LAUNCH_URL` in a CI variable and your destructive suite is pointed at the live site. Tag every scenario with the environments it is safe to run in, so the protection lives in the test itself instead of in everyone remembering to be careful.

### Pattern from Varbase

```gherkin
@local @development
Scenario: Admin can disable a user
  Given I am a logged in user with the "Webmaster" user
  And I am on "/admin/people"
  When I click "Edit" in the "Normal user" row
  And I check "Blocked"
  And I press "Save"
  Then I should see "The changes have been saved"
```

This test never runs against staging or production, by construction. The reverse -- read-only smoke tests -- carry all four environment tags.

### Recipe

For every new scenario, ask: which environments can this run in safely? Tag accordingly. Wire CI to filter by environment:

```bash
ENV=production cucumber-js --tags "@$ENV"
```

Two details make this fail safe rather than merely tidy. Tags are an allow-list: under a positive tag filter, a scenario carrying no environment tag never runs in any pipeline -- so a forgotten tag means a skipped test, not a destructive one loose on production. And the target site comes from `LAUNCH_URL` -- webship-js reads it into `worldParameters.launchUrl`, defaulting to `http://localhost:8080` -- so one pipeline definition serves every environment by swapping two variables: `ENV` picks *which* scenarios run, `LAUNCH_URL` picks *where* they run. The pairing is the protection: the production pipeline physically cannot select a `@local`-only scenario, no matter whose finger slips.

**At scale:** this is how one suite serves a whole fleet. At Vardot the CI template is identical across projects; only `ENV` and `LAUNCH_URL` differ per pipeline, and both live in CI variables rather than in the repo. Nobody edits a config file to point tests at a new environment -- which also means nobody fat-fingers one. And when a security review asks "what stops the destructive suite from reaching production?", the answer is a tag expression in version control, not a promise that everyone stays careful.

---

## Recipe 528: The "Logged In As" Custom Step

### Lesson
Open a feature file from the early Varbase suite and the first six lines of every scenario were identical: go to the login page, fill in the name, fill in the password, press the button, wait. Multiply by a few hundred scenarios and you are paying ten seconds of login per scenario and forcing every reviewer to re-read the same six lines until their eyes learn to skip them -- and eyes that skip the setup also skip the one scenario where the setup is subtly wrong, which is the one hiding the bug.

### Pattern from Varbase
Varbase ships a single custom step that handles the entire login flow:

```js
Given(/^I am a logged in user with( the)*( username)* "([^"]*)?"( user)*$/,
  async function(_, __, username, ___) {
    const user = USERS[username];
    await this.page.goto('/user/login');
    await this.page.fill('#edit-name', user.name);
    await this.page.fill('#edit-pass', user.pass);
    await this.page.click('#edit-submit');
    await this.page.waitForSelector('.toolbar-user');
  });
```

Then every feature uses one line:

```gherkin
Given I am a logged in user with the "Webmaster" user
```

### Recipe

Build one custom step per long, repeated flow. Wrap the dance once; use the readable name everywhere. Login is the most common -- but the same pattern applies to "I have a published article", "I have an active reservation", "the cart contains 3 items". Each becomes a one-liner that reads like a domain sentence.

Two implementation details in the Varbase step are worth stealing. The regex's optional groups (`( the)*`, `( username)*`) let one definition match several natural phrasings, so authors never have to memorise a single blessed wording. And the final `waitForSelector('.toolbar-user')` defines success as *an element that only exists when you are logged in* -- the step waits for evidence, not for time. When login gets slower, the step still passes; when login silently fails, the step fails right there instead of three steps later in some unrelated assertion.

If even one scripted login per scenario is too slow, webship-js 2.0 ships auth-state steps that save and restore Playwright's `storageState`: log in once, save the state, and every later scenario restores cookies and local storage in milliseconds. The custom step stays as the readable front door; the storage-state steps are the fast lane behind it.

**At scale:** this step is the reason the login line reads identically on every Vardot project. The definition ships in the shared package; only the fixture file mapping labels like "Webmaster" to real credentials differs per site. A QA engineer who rotates across three projects in a week types the same sentence on all three -- and when one client adds a 2FA screen, the fix lands in one step file, not in forty copies of a login dance.

---

## Recipe 529: Test User Fixtures

### Lesson
The staging password rotates on a Tuesday and forty scenarios go red at once. Someone greps the feature files and finds the old password hard-coded in thirty places, each pasted from the scenario before it. The fix eats an afternoon of find-and-replace -- and the same afternoon repeats at the next rotation, because the data lives *in* the scenarios instead of behind them.

### Pattern from Varbase

```js
// fixtures/users.js
module.exports = {
  Webmaster: {
    name: 'webmaster',
    pass: process.env.WEBMASTER_PASS || 'dD.123123ddd',
    roles: ['administrator'],
  },
  'Normal user': {
    name: 'normal_user',
    pass: process.env.NORMAL_USER_PASS || 'dD.123123ddd',
    roles: ['authenticated'],
  },
  ...
};
```

The custom step from Recipe 528 looks up the user by label:

```js
const user = USERS[username];
```

### Recipe

Maintain a single fixture file for test data. Use environment variables for secrets. Reference fixtures by label in features. When a password changes, update one line. When a new test user is needed, add one record. Features never change.

Notice the shape of the fixture: the key is a human label ("Webmaster", "Normal user"), the value is machine detail (account name, password, roles). Features speak only the label. That one level of indirection is what lets the suite survive credential rotation, environment differences, even a rename of the underlying account -- the Gherkin sentence `I am a logged in user with the "Webmaster" user` outlives all of it. And the `process.env.WEBMASTER_PASS || 'dD.123123ddd'` pattern gives you a safe local default and a CI override in the same line: developers run against a known local password, pipelines inject the real secret, and no production credential ever lands in the repo.

**At scale:** standardise the labels, not the credentials. Every Vardot project has a "Webmaster", a "Content editor", and a "Normal user" -- different real accounts on every site, same labels everywhere. That convention makes feature files portable between projects, lets onboarding docs say "use the Webmaster fixture" instead of circulating passwords, and gives the security team exactly one answer to "where do test credentials live?": in each pipeline's secret store, injected as env vars.

---

## Recipe 530: Multi-Step Scenarios That Mirror Real Workflows

### Lesson
The create form has a test. The edit form has a test. Both pass, and the bug ships anyway -- because it only appears when an editor creates an article, then edits it, then unpublishes it, and the unpublish drops the revision the edit just made. The bugs that matter happen at workflow seams, and a suite built only from single-action scenarios never crosses a seam.

### Pattern from Varbase

```gherkin
@content-management @local @development
Scenario: Article workflow end-to-end
  Given I am a logged in user with the "Content editor" user
  And I am on "/node/add/article"
  When I fill in the field "Title" with "My Article"
  And I fill in the WYSIWYG field "Body" with the "Lorem ipsum..."
  And I press "Save and publish"
  Then I should see "Article My Article has been created"
  When I click "Edit"
  And I fill in the field "Title" with "My Updated Article"
  And I press "Save and keep published"
  Then I should see "Article My Updated Article has been updated"
  When I click "Edit"
  And I select "Unpublished" from "Status"
  And I press "Save"
  Then I should see "has been updated"
  And the path should be "/node/X/edit"
```

The scenario is long, but it covers a real lifecycle: create → edit → unpublish.

### Recipe

When in doubt, mirror what a human would do. A 20-step scenario that follows a user journey catches bugs three single-step scenarios would miss. The trade-off is failure isolation -- when a 20-step scenario fails, you need the report to tell you which step. Webship-js solves that twice over: on failure it screenshots automatically (`WEBSHIP_SCREENSHOT_ON_FAILED` defaults to true), and after every state-changing step its AfterStep hook runs `smartSettle(page, 1500)`, so step 14 never fails because step 13's AJAX was still in flight. Long journeys are exactly where that auto-settle earns its keep -- the failure you see is the failure that actually happened.

**Pro Tip:** tag your longest journeys `@screenshots`. That flips on every-step capture for just those scenarios, and when one fails you can flip through the screenshot trail like a flipbook and watch the workflow break in slow motion.

---

## Recipe 531: The Background Block

### Lesson
Five scenarios in a feature, and each one opens with the same two lines of setup. The duplication looks harmless until the admin path changes and the fix is a five-line diff that should have been one -- or until a new scenario quietly gets the setup slightly wrong and spends six months testing a different page than its siblings.

### Pattern from Varbase

```gherkin
Feature: Admin content management

  Background:
    Given I am a logged in user with the "Webmaster" user
    And I am on "/admin/content"

  Scenario: Filter by published status
    When I select "Published" from "Status"
    And I press "Filter"
    Then I should see at least one published article

  Scenario: Filter by author
    When I fill in "Authored by" with "webmaster"
    And I press "Filter"
    Then I should see articles authored by "webmaster"
```

`Background:` runs before every scenario in the file. Two scenarios, one shared setup, no repetition.

### Recipe

When two or more scenarios in a feature share the first 2-3 steps, lift them into `Background:`. When they share more than 5 steps, consider whether they really belong in the same feature -- or whether a feature split would read better.

Remember what `Background:` is not: it is not run-once setup. It executes before *every* scenario in the file, so a ten-scenario feature pays that login ten times. That is the correct default -- scenarios stay independent and can run in any order, or in parallel -- but if the repeated login starts to dominate runtime, pair the pattern with webship-js's auth-state steps: log in once, save the `storageState`, and let the Background restore it in milliseconds instead of replaying the form. Same isolation, a fraction of the cost.

---

## Recipe 532: Tag-Driven Test Selection

### Lesson
Your commit changes one stylesheet, and CI makes you wait forty minutes while 500 scenarios re-verify the password-reset flow. By Thursday, developers are merging without waiting for green -- and once people stop waiting for the suite, the suite has stopped doing its job. A suite that can only run all-or-nothing will eventually run as nothing.

### Pattern from Varbase

```bash
# Full suite (nightly)
cucumber-js

# Smoke only (every commit)
cucumber-js --tags @smoke

# JavaScript-dependent scenarios only (when JS changes)
cucumber-js --tags @javascript

# Skip flaky scenarios (during a bug hunt)
cucumber-js --tags "not @flaky"

# Run only a specific area (during local development)
cucumber-js --tags @content-management
```

### Recipe

Adopt a tag taxonomy from day one. Suggested:

- Environment tags: `@local @development @staging @production`
- Speed tags: `@smoke @full @regression`
- Area tags: `@user-management @content @api ...`
- Status tags: `@flaky @wip @broken`

Wire each tag to a CI workflow. The result is a test suite that's responsive at every commit cadence -- not all-or-nothing.

One thing the taxonomy gives you that is easy to miss: in webship-js, tags don't just *select* scenarios, they change how scenarios *run*. `@screenshots` switches on per-step capture, `@video` forces a recording, `@js-fail` makes any console error fail the scenario, and `@no-video` and `@js-off` switch those back off. So `cucumber-js --tags @flaky` doesn't just isolate your problem children -- tag those same scenarios `@video @screenshots` and the next run comes back with full forensics, no config edits anywhere.

**At scale:** the taxonomy is only useful if it is the *same* taxonomy everywhere. Vardot fixed the tag names once, org-wide, and every project inherits them -- which means one CI template serves every repo, a QA lead can ask "show me @smoke across all thirty sites" and get a real answer, and an engineer borrowed from another team already knows what `@flaky` means before they open the README. The worst version of tagging is each project inventing its own dialect.

---

## Recipe 533: The Three-File Setup

### Lesson
The third project you onboard is where layout drift starts to bite: the first repo keeps step definitions in `tests/steps/`, the second in `features/support/`, the third somewhere only its author remembers. The new hire's first PR puts files in the wrong directory because there was no right one. A real Varbase test project has exactly three top-level pieces, and every project keeps them in the same place:

```
cucumber.js                  ─ Cucumber runtime config
playwright.config.ts         ─ Playwright runtime config
tests/
  step-definitions/
    *.steps.js               ─ shared step files
  features/
    01-*/
    02-*/
    ...
  hooks/
    world.js                 ─ shared World class
    setup.js                 ─ before-all
    teardown.js              ─ after-all
```

Everything else is project-specific.

### Recipe

Don't over-architect. Three top-level config files, one step-definitions directory, one features directory organised by area. That's it. Any extra directory layer past the second adds friction for new contributors.

This layout is not just convention -- it is what webship-js's defaults already expect. The stock `cucumber.js` profile loads steps from `tests/step-definitions/**/*.js`, features from `tests/features/**/*.feature`, and writes the JSON report to `tests/reports/cucumber_report.json`, with a 45-second step timeout. The stock `playwright.config.ts` reads `BROWSER`, `HEADLESS`, and `SLOW_MO` from the environment. Keep the standard shape and your per-project configuration shrinks to a handful of env vars; fight the shape and every project grows a bespoke config somebody has to maintain forever.

**At scale:** identical layout is an onboarding feature. At Vardot, anyone who has touched one suite can open any other repo and know where the features, steps, and reports live before the clone finishes. Project differences are expressed in env vars -- `LAUNCH_URL`, `BROWSER`, credentials -- never in moved directories. When something genuinely must differ structurally, write down why, because that difference will tax every engineer who visits the repo after you.

---

## Recipe 534: The Site-Status Sentinel

### Lesson
The disk fills up on a Friday night, the files directory turns read-only, and nothing visibly breaks -- pages render, login works, the suite stays green. Monday morning an editor tries to upload an image and the support queue lights up. Drupal's status report had been flagging the problem all weekend; nobody was looking at it. That report aggregates DB connectivity, file permissions, module health, and update availability into one page -- which makes it one page a robot should read every night.

### Pattern from Varbase

```gherkin
@smoke @local @development @staging @production
Scenario: System status report is clean
  Given I am a logged in user with the "Webmaster" user
  And I am on "/admin/reports/status"
  Then I should not see "Errors"
  And there should be no JavaScript errors
```

This single scenario catches an enormous class of bugs: a missing module file, a writable directory turning read-only, a database connection drifting, an expired SSL cert on a back-end service.

### Recipe

Find the equivalent in your stack. WordPress: `/wp-admin/site-health.php`. Laravel: `/up`. Rails: `/rails/info/properties`. Express: a custom `/_health` endpoint. Add a smoke scenario that asserts the dashboard reports green. Run it as part of every CI run.

The scenario is two assertions, but each is wider than it looks. `I should not see "Errors"` reads the page the platform itself uses to summarise dozens of subsystem checks -- so you inherit every check the platform adds in future versions, for free. And `there should be no JavaScript errors` rides on webship-js's console capture, which has been listening on every step all along (`WEBSHIP_JS_ERROR_MODE` defaults to `warn`); the step turns that quiet listening into a hard assertion. One scenario, dozens of failure modes covered.

**At scale:** one sentinel per site, run nightly, becomes a fleet health board. Generate the report with metadata -- `bin/generate-reports.js --env staging --app-version 10.1.3` -- and "are all thirty sites healthy this morning?" stops being thirty SSH sessions and becomes one glance down a list. This is the cheapest monitoring you will ever ship, because the check already existed inside the platform; you just pointed a scenario at it.

---

## Recipe 535: The "It Doesn't Crash" Scenario

### Lesson
A deploy goes out with a broken template on the search page. Every test passes -- because every test starts at the homepage and follows the happy paths, and none of them ever visits `/search`. The first person to find the white screen is a visitor, the second is your client, and the third, finally, is you. Half of acceptance testing is "this page renders without exploding." Don't underestimate it.

### Pattern from Varbase

```gherkin
@smoke
Scenario Outline: Page <path> renders without errors
  Given I am on "<path>"
  Then there should be no JavaScript errors
  And I should not see "Drupal\\Core\\Database\\DatabaseExceptionWrapper"
  And I should not see "Fatal error"
  And I should not see "PHP Fatal Error"

  Examples:
    | path                  |
    | /                     |
    | /node                 |
    | /admin                |
    | /admin/structure      |
    | /admin/reports        |
    | /search               |
```

One outline, six paths, six guaranteed-rendering pages -- and adding a seventh costs one table row.

### Recipe

Write a smoke outline that visits every critical path. It's not exciting but it catches deploys that broke routing, asset compilation, or template parsing. Run it before any complex scenarios -- if the basics fail, the rest is noise.

The outline earns more than it shows. Each `Given I am on "<path>"` runs through smartSettle -- the page must reach `domcontentloaded`, finish its fetch/XHR traffic, drain its pending timers, and hold the DOM quiet for 250ms before the step passes -- so "renders" means *finished rendering*, not "returned some HTML". The negative assertions then sweep the settled page for the signatures your stack prints when it dies. Swap in your own: Rails' "We're sorry, but something went wrong", Laravel's "Whoops", a React error boundary's fallback text. The error strings are the only framework-specific part of the recipe.

---

## Recipe 536: The Permission Matrix

### Lesson
A permissions refactor lands, and in review someone asks the simple question: "can a Content editor still reach /admin/people?" Nobody knows. Answering by hand means logging in as four different users and clicking through every admin path -- so nobody does, and the regression ships. "Who can access what" is the most regression-prone area in any CMS precisely because checking it manually is tedious enough that humans skip it.

### Pattern from Varbase

```gherkin
Scenario Outline: <role> access to <path>
  Given I am a logged in user with the "<role>" user
  When I am on "<path>"
  Then I should <expectation>

  Examples:
    | role            | path                | expectation        |
    | Webmaster       | /admin/people       | not see "Access denied" |
    | Site editor     | /admin/people       | see "Access denied"     |
    | Content editor  | /admin/people       | see "Access denied"     |
    | Normal user     | /admin/people       | see "Access denied"     |
    | Webmaster       | /admin/structure    | not see "Access denied" |
    | Site editor     | /admin/structure    | not see "Access denied" |
    | Content editor  | /admin/structure    | see "Access denied"     |
    | Normal user     | /admin/structure    | see "Access denied"     |
```

One outline, four roles, every admin path. A regression in `permissions.module` lights up immediately.

### Recipe

Build a single Permission Matrix feature per role-aware area. Include both positive (allowed) and negative (denied) cases. Update the matrix whenever roles or permissions change. A 20-row table beats 20 separate scenarios for maintenance.

The negative rows matter more than the positive ones. A role that *gains* access you didn't intend is a security bug, and it is invisible to manual testing because nothing looks broken. The Examples table also turns the access policy into a reviewable artifact: a product owner can read it top to bottom and confirm it matches intent without reading a line of code, and a permissions change shows up in the diff as changed table rows -- the clearest review a security-sensitive change can get.

**At scale:** keep the matrix shape identical across projects and it becomes your audit answer. When a government client asks "show us who can administer users", the response is the feature file plus its latest green report -- the spec and the proof in one place. A new role on one site means new rows, not new scenarios; the outline machinery is already there, and so is the habit of reading it.

---

## Recipe 537: The Audit Trail Test

### Lesson
The incident review is the worst possible moment to learn your audit log stopped recording two releases ago. Audit trails fail silently by nature -- no user complains, no page breaks, the table just quietly stops growing. Sensitive systems need an audit trail, and the audit trail itself needs a test, because the day you consult it is the day you cannot afford it to be empty.

### Pattern from Varbase

```gherkin
@audit @local @development
Scenario: Failed admin login is recorded
  Given I am on "/user/login"
  When I fill in "username" with "webmaster"
  And I fill in "password" with "wrong-password"
  And I press "Log in"
  Then I should see "Unrecognized username or password"

  When I am a logged in user with the "Webmaster" user
  And I am on "/admin/reports/audit-trail"
  Then I should see "Authentication failure"
  And I should see "webmaster" in the most recent audit row
```

The first half creates an audit event; the second half asserts the event was recorded.

### Recipe

For every audit-worthy action in your app, write a paired test: action + verification. The act of writing the test forces you to confirm the audit pipeline is wired correctly. Skip the test and the audit log will silently break on the day you need it most.

Note the structure: the scenario *causes* an event as one identity, then *verifies* it as another. The failed login must come from a clean session -- if a previous scenario could have left cookies behind, open with webship-js's `Given I am an anonymous user`, which clears cookies before navigating, so the failure you record is genuinely an anonymous one. Self-contained cause and effect, no dependence on whatever happened to be in the log already. That is what makes this a test instead of a hope.

---

## Recipe 538: The JSON-API Contract Test

### Lesson
The mobile team builds against your JSON-API. Six months later a module update renames a field, their app breaks on launch, and you find out from their bug report -- filed after their users filed theirs. Nothing in your UI suite noticed, because your UI never reads that field. A REST API contract is a contract whether you wrote it or someone else did, and the side that can test it cheapest is yours.

### Pattern from Varbase

```gherkin
@api @local @development @staging
Scenario: JSON-API exposes articles
  Given I am authenticating as "admin_api" with "API_TOKEN" password
  When I send a GET request to "/jsonapi/node/article"
  Then the API response code should be 200
  And the JSON response should have property "data"
  And the JSON response should have property "links.self"
  And the JSON response should not contain "errors"
```

### Recipe

For every JSON-API resource, write a contract test that asserts: response code, the presence of `data`, the presence of canonical `links`, and the absence of `errors`. If the contract drifts, the test fails. Stakeholders who depend on the API get a heads-up before they file a bug.

The block above shows the Behat-era phrasing from the Varbase suite; webship-js 2.0 runs API scenarios through axios with no browser involved, which makes them the fastest tests you own. The modern shape: `Given the API base URL is "..."` to point at the service, `Given I set header "H" with value "V"` for the auth token, then `Then the API response code should be 200` plus JSON-pointer assertions like `Then the JSON response should have "/data" equal to value` -- pointers reach into nested structures without brittle string matching. Either way, assert the *shape* (data, links, no errors), not the full payload: contract tests that pin every byte break on every harmless change, and tests that cry wolf get ignored.

**At scale:** contract tests are the cheapest cross-team insurance you can buy. Tag them `@api`, run them on every backend deploy -- no browser, so they cost seconds -- and publish the report where the consuming teams can see it. At Vardot, the JSON-API contract checks are what let a distribution update roll out to client sites without a round of "did this break your integration?" emails.

---

## Recipe 539: The Tour Test

### Lesson
A designer reorders the admin toolbar, ships it, and the product tour now anchors its third tooltip to empty space. No error fires, no test fails, no log line appears -- a tour pointing at nothing is, to every machine involved, working as intended. You learn about it months later from a customer-success call, after every new user in between took a broken first-run experience as their introduction to your product.

### Pattern from Varbase

```gherkin
@tour
Scenario: Admin tour runs end-to-end
  Given I am a logged in user with the "Webmaster" user
  When I am on "/admin"
  And I click "Tour"
  Then I should see "Welcome to Varbase" in the tour
  When I click next button in tour
  Then I should see "Manage content" in the tour
  When I click next button in tour
  Then I should see "Manage users" in the tour
  ...
```

Note `click next button in tour` is a custom step -- it knows the tour-library's selector pattern.

### Recipe

If your product ships a tour, add a tour test. The cost is low; the value spikes the day a designer reorders the page and the tour starts pointing at the wrong element.

The custom step is the load-bearing decision here. `click next button in tour` hides the tour library's selector pattern inside exactly one step definition -- so when you swap tour libraries, or the library renames its CSS classes, you fix one line in one file and every tour scenario keeps reading the same. And write the assertions against the tooltip *text* the user reads, not the library's DOM structure: the text is the part your product team actually owns.

---

## Recipe 540: The Persistent-Login Test

### Lesson
No developer manually tests "Remember me" -- doing it honestly means closing the browser and coming back tomorrow, or thirty days from now. So the checkbox ships, the persistent cookie silently never gets set, and the bug lives for a quarter until a pattern surfaces in support tickets: "the site keeps logging me out." Easy to write, impossible to test by clicking around. Exactly the kind of feature automation exists for.

### Pattern from Varbase

```gherkin
@auth @local @development
Scenario: Persistent login survives session expiry
  Given I am on "/user/login"
  And I fill in "username" with "normal_user"
  And I fill in "password" with "dD.123123ddd"
  And I check "Remember me"
  And I press "Log in"
  Then I should see "Welcome"
  When the cookie "session" is removed
  And I reload the page
  Then I should still see "Welcome"   # persistent cookie holds the session
```

### Recipe

If "Remember me" exists, test it. The bug is usually that the persistent cookie wasn't set, and only an automated test catches it because no developer manually waits 30 days to test their own login.

The trick in the scenario is simulating time instead of waiting for it: delete the short-lived session cookie, reload, and see whether the persistent token re-authenticates you. Make the test sharper by asserting the cookie exists *before* you rely on it -- webship-js ships twelve cookie matchers, so `Then a cookie with the name "session" should exist` (or a name-containing variant, for cookies with generated suffixes) confirms the persistent cookie was actually written at login. That splits any failure into two clean diagnoses: cookie never set, or cookie set but ignored on the way back in.

---

## Recipe 541: The Password-Reset Flow

### Lesson
A deploy swaps the mail backend, the password-reset email silently stops sending, and the users who hit the bug are by definition the ones who cannot log in to complain. They retry, fail, and leave. Password reset is the single most-tested feature in any auth system and the single most-broken after a deploy goes wrong -- because it spans the app, the queue, and the mail service, and most suites only test the first third.

### Pattern from Varbase

```gherkin
@auth @local @development
Scenario: Request a password reset
  Given I start recording network requests
  And I am on "/user/password"
  When I fill in "Username or email address" with "normal_user"
  And I press "Submit"
  Then I should see "Further instructions have been sent"
  And a POST request to "**/user/password" should have been made
```

For the email half, hook into a mail-catching service (MailHog locally) and assert the email arrived:

```gherkin
  When I am on "http://localhost:8025"
  Then I should see "Replacement login information for normal_user"
```

### Recipe

Test the request half always. Test the email-receipt half whenever you have a local mail catcher. The full password-reset flow is one of the highest-value tests in any auth system.

---

## Recipe 542: The Content-Editor Workflow

### Lesson
WYSIWYG editors have a thousand small ways to break. The fix for one breaks two more. Test the editor as a workflow, not a widget.

### Pattern from Varbase

```gherkin
@content-management
Scenario: Editor preserves content across save and edit
  Given I am a logged in user with the "Content editor" user
  And I am on "/node/add/article"
  When I fill in the field "Title" with "My Article"
  And I fill in the WYSIWYG field "Body" with the "Paragraph one.\n\nParagraph two."
  And I press "Save and publish"
  Then I should see "My Article"
  And I should see "Paragraph one."
  And I should see "Paragraph two."
  When I click "Edit"
  Then the WYSIWYG field "Body" should contain "Paragraph one."
  And the WYSIWYG field "Body" should contain "Paragraph two."
```

The second half is the kicker: the editor must round-trip content through save and re-edit. Many editors lose newlines or convert paragraph tags on save.

### Recipe

For every rich-text editor, write a round-trip test. The bug it catches is the one your customers complain about most.

---

## Recipe 543: The Image-Upload Workflow

### Lesson
File uploads sit at the seam between browser, app server, and storage backend. The seam leaks regularly.

### Pattern from Varbase

```gherkin
@media @local @development
Scenario: Image upload generates derivatives
  Given I am a logged in user with the "Content editor" user
  And I am on "/media/add/image"
  When I attach the file "fixtures/test-image.jpg" to "Image"
  And I fill in "Name" with "Test Image"
  And I press "Save"
  Then I should see "Test Image"
  And the path should match "/media/[0-9]+"
  When I am on "/admin/content/media"
  Then I should see "Test Image"
  And I should see "test-image.jpg"
```

Followed by the derivatives check:

```gherkin
  When I am on "/sites/default/files/styles/large/public/test-image.jpg"
  Then the response status code should be 200
```

### Recipe

Test uploads end-to-end. Don't stop at "the file uploaded" -- assert the derivative URLs serve correctly, the metadata persisted, and the asset is reachable from the public URL.

---

## Recipe 544: The "Masquerade" Pattern

### Lesson
Admin tools that switch users (masquerade, switch-user, sudo-to) need a test because they're exactly the kind of feature that breaks without anyone noticing.

### Pattern from Varbase

```gherkin
@admin @local @development
Scenario: Admin masquerades as a normal user
  Given I am a logged in user with the "Webmaster" user
  And I am on "/admin/people"
  When I click "Masquerade" in the "Normal user" row
  Then I should see "You are now logged in as normal_user"
  When I am on "/user"
  Then I should see "normal_user"
  When I click "Unmasquerade"
  Then I should see "webmaster"
```

The pattern: switch in, verify, switch out, verify return.

### Recipe

For any "switch user" admin tool, write a switch-and-return scenario. Test what happens if the user logs out mid-masquerade (the parent session should resume or end cleanly). Test that masquerade-as cannot bypass MFA on the target user.

---

## Recipe 545: The Cache-Invalidation Test

### Lesson
"Cache invalidation is one of two hard things in computer science." It's also one of two regression hotspots.

### Pattern from Varbase

```gherkin
@cache @local @development
Scenario: Updating an article invalidates the listing page
  Given I am a logged in user with the "Content editor" user
  When I am on "/blog"
  Then I should see "Old Title"
  When I am on "/node/42/edit"
  And I fill in the field "Title" with "New Title"
  And I press "Save and keep published"
  When I am on "/blog"
  Then I should see "New Title"
  And I should not see "Old Title"
```

### Recipe

Identify the listing pages that should refresh when a piece of content changes. Write a scenario: change the content, navigate to the listing, assert the new state. If the cache layer holds a stale value, this scenario fails.

---

## Recipe 546: The Cross-Browser Smoke

### Lesson
A test that passes in Chromium and fails in WebKit is a real bug. Catch it before users do.

### Pattern from Varbase
The Varbase CI matrix runs the smoke suite across three browsers:

```yaml
strategy:
  matrix:
    browser: [chromium, firefox, webkit]
```

With `BROWSER=${{ matrix.browser }}` plumbed through to `playwright.config.ts`.

### Recipe

Run smoke-tagged scenarios across all three Playwright browsers. Run the full suite on chromium only -- it's the cheapest and catches most issues. Reserve cross-browser for smoke and any known-finicky area (rich-text, drag-and-drop, custom video controls).

---

## Recipe 547: The Update-Path Test

### Lesson
A Drupal update path can break in ways unit tests can't detect. The only protection is to actually run the update and see what survives.

### Pattern from Varbase

```gherkin
@update-path @local
Scenario: Update from 10.0 to 10.1 preserves content
  Given I am running Varbase 10.0
  And there are 5 published articles
  When I update to Varbase 10.1
  And I am on "/blog"
  Then I should see 5 published articles
  And I should not see "There was an error executing the update"
```

The first two `Given` steps are custom -- they shell out to install a known version, then load fixture content.

### Recipe

For any product with an update path (CMS, framework, library), build a regression scenario for each major version transition. The cost is low; the value is "we caught the migration bug before customers ran it on Friday at 5pm."

---

## Recipe 548: The Performance-Regression Sentinel

### Lesson
A test that passes too slowly is also a kind of bug. Catch large regressions in performance.

### Pattern from Varbase

```gherkin
@performance @smoke
Scenario: Homepage loads in under 2 seconds
  Given I start the performance timer
  When I am on "/"
  And I stop the performance timer
  Then the elapsed time should be under 2000 milliseconds
```

The performance steps are custom; the implementation is straightforward (record `performance.now()` at start and end).

### Recipe

Add a small set of performance-sentinel scenarios. They shouldn't replace real performance testing -- but they catch the 10x regression caused by a debug logger left on in production.

---

## Recipe 549: The "No Broken Images" Sweep

### Lesson
A page with an `<img>` whose `src` returns 404 is broken from the user's perspective. The page still renders, so no other test catches it.

### Pattern from Varbase

```js
Then(/^there should be no broken images$/, async function() {
  const broken = await this.page.evaluate(() => {
    return [...document.images]
      .filter(img => !img.complete || img.naturalWidth === 0)
      .map(img => img.src);
  });
  if (broken.length) {
    throw friendly(`Broken images:\n${broken.join('\n')}`);
  }
});
```

```gherkin
@smoke
Scenario: Homepage has no broken images
  Given I am on "/"
  Then there should be no broken images
```

### Recipe

Add the "no broken images" assertion to every smoke scenario. The 10 lines of custom-step code catch one of the most common visible bugs in any visual CMS.

---

## Recipe 550: The Pre-Release Suite

### Lesson
Before a release, you want one fast, comprehensive sweep that confirms the basics. Build it as a tagged collection.

### Pattern from Varbase

```bash
# Pre-release check
cucumber-js --tags "@smoke and @local"
```

Runs in 5 minutes. Covers the smoke tests, the basic permission matrix, and the most-trafficked workflows. If green, the release is safe to cut.

### Recipe

Define what "safe to release" means in your project and tag the scenarios that prove it. Have engineering, QA, and product agree on the set. Run it before every release. When it goes red, halt.

---

## Recipe 551: The "Tests Run In Parallel" Lesson

### Lesson
A test suite that runs scenarios serially scales as O(N). One in parallel scales as O(N/cores). Big difference at 500 scenarios.

### Pattern from Varbase
The Varbase CI matrix shards scenarios across 4 parallel jobs by feature directory. Each job runs ~125 scenarios in 7-9 minutes. Total wall time: 9 minutes for the full suite vs 36 minutes serial.

### Recipe

When your suite passes 50 scenarios, look at parallelisation. Cucumber-js's `--parallel N` flag is the cheapest entry point. CI-level parallelism (matrix sharding) scales further but needs careful test-data isolation.

---

## Recipe 552: The "One Scenario, One Idea" Rule

### Lesson
A scenario that tests three things will, on failure, give you three guesses about which thing broke. A scenario that tests one thing will tell you exactly.

### Pattern from Varbase
Every Varbase scenario answers one question. The scenario name reads as the question:

```
Scenario: User can request a password reset
Scenario: Password reset email contains a working link
Scenario: Reset link expires after 24 hours
```

Three scenarios, three behaviours. The previous form was:

```
Scenario: Password reset works end-to-end   ← too big
```

Three failures merged into one ambiguous one.

### Recipe

When the scenario name doesn't fit on one line, split. When the scenario has more than ten steps, ask whether two scenarios would be clearer.

---

## Recipe 553: The DAMP Principle for Tests

### Lesson
Production code is DRY (Don't Repeat Yourself). Test code is DAMP (Descriptive And Meaningful Phrases). The two have different rules.

### Pattern from Varbase
Varbase tests repeat short fragments deliberately:

```gherkin
And I press "Save"
Then I should see "The changes have been saved"
```

This duplicates across scenarios. We could DRY it into `When I save the form`. We don't, because the explicit version reads better in feature reports.

### Recipe

Repeat short, readable phrases across scenarios. Extract only when the extraction adds genuine clarity (`I am a logged in user with the "X" user`). The cost of repetition is two extra lines per scenario; the benefit is readability for non-engineers.

---

## Recipe 554: Test Data as Code

### Lesson
Test data living in JSON files is fine until you need to derive one record from another, or compute a timestamp, or branch by environment. Then you want code.

### Pattern from Varbase

```js
// fixtures/articles.js
module.exports = {
  publishedArticle: (overrides = {}) => ({
    title: `Article ${Date.now()}`,
    status: 1,
    body: 'Lorem ipsum',
    ...overrides,
  }),
};
```

In a step:

```js
const article = articleFixtures.publishedArticle({ title: 'My Article' });
```

The factory function makes derivation cheap.

### Recipe

When fixture JSON starts repeating itself or needs computed values, move to factory functions. The pattern scales further than you'd expect -- many large suites end up with a single `factories.js` per domain object.

---

## Recipe 555: The Lessons-Learned Convention

### Lesson
Every bug you fix is a lesson. Capture it as a test before you fix the bug.

### Pattern from Varbase
Varbase's `tests/features/regressions/` directory has a feature per closed bug:

```
regressions/
  bug-1284-empty-coupon.feature
  bug-1456-stale-cache-after-publish.feature
  bug-1502-permission-cache-leak.feature
```

Each feature has one scenario: the smallest possible reproduction of the bug, plus the expected behaviour after fix.

### Recipe

Adopt the convention. The next time someone files a bug, write the reproduction scenario before writing the fix. The test goes red. You fix the bug. The test goes green. The bug never returns -- which is exactly what regression suites are for.

---

## Recipe 556: The Webship-js Origin Story

### Lesson
We did not set out to build webship-js. We set out to test Varbase. Webship-js is what we extracted from the Varbase test rig once we noticed we were rebuilding the same thing every time a new project started.

The pattern recurred:

1. New project starts. Cucumber + Behat + Selenium gets stood up.
2. Step definitions are written, copy-pasted from the last project.
3. After three months, the step definitions diverge silently.
4. Bugs in one project's step definitions don't get fixed in the other.

The fix was to make the step definitions a package: install once, share across projects. Webship-js is that package.

### Recipe

If you find yourself rebuilding the same test rig across projects, extract it. The cost of extraction is two days. The cost of not extracting is a decade of divergence.

---

## Recipe 557: The Drupal-First, Framework-Agnostic Path

### Lesson
Webship-js was born from a Drupal test suite. Its first hundred features were Drupal-specific. But every step we wrote was about user behaviour, not about Drupal -- "log in", "fill in a field", "save a node". When we generalised to non-Drupal projects, almost everything carried over.

### Pattern
Look at the step names. None of them mention Drupal. The Drupal-ness lives in step definitions and fixtures, not in features.

### Recipe

When you write a step, ask: does the step name describe user behaviour or framework behaviour? If it's framework behaviour, hide it behind a user-behaviour name. Your feature files become portable across stacks.

---

## Recipe 558: The Vardot Stack as Reference Implementation

### Lesson
We use webship-js on every Vardot project. The Vardot stack is, in effect, the reference implementation -- the codebase we hit every edge case against.

If you want to see webship-js in production, look at:

- The Varbase 10.1.x test suite (Drupal 10)
- The Varbase 11.0.x project test suite (Drupal 11)
- Vardot client projects (private, but the patterns are documented here)

### Recipe

When you adopt webship-js, treat the Varbase test suite as your reading room. The patterns we ship in this book are the ones we'd adopt in any new project, distilled from years of running Varbase tests. Adapt, don't copy verbatim -- but the shape will help you start.

---

## Recipe 559: What We Got Wrong (And Then Fixed)

### Lesson
Three things we got wrong in the early days:

1. **Selenium-style waits everywhere.** Every step had a manual `wait for AJAX` after it. Tests were slow and brittle. We added web-first assertions in webship-js ~2.0; now `wait for AJAX` is rare in new tests.
2. **JSON fixtures, not factories.** Test data was 30 huge JSON files that became impossible to keep in sync. Moving to factory functions cut maintenance to a fraction.
3. **One giant feature per area.** A 60-scenario feature was unreadable. Splitting by behaviour (one feature per workflow, not one per area) made reviews 5x faster.

### Recipe

If your suite resembles any of those three early-Varbase patterns, plan a migration. The cost is real but bounded; the payoff is permanent.

---

## Recipe 560: The Long Game

### Lesson
A test suite is a long-term asset. The patterns that matter most aren't the clever ones; they're the boring ones. Naming conventions. Tag taxonomies. Stable fixtures. Predictable file structure. Small scenarios. Readable failures.

### Pattern from Varbase
Ten years in, the suite that started as one developer's Behat experiment has run on every CI run for thousands of releases. The patterns that survived are in this chapter. The ones that didn't -- single-file mega-features, mocked DBs, complex inheritance hierarchies in step definitions -- got refactored out years ago.

### Recipe

Build for the version of yourself who maintains the suite three years from now. They will not remember the clever pattern. They will, however, appreciate the boring one that still works.

---

We learned a lot by experimenting and working on Varbase. Every recipe in this book carries some fingerprint of that experience -- the patterns kept, the patterns discarded, and the surprises that taught us how acceptance testing actually works in production. If you only adopt one chapter into your own practice, make it this one.

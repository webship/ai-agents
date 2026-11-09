# Chapter 4: Your First Real Test Suite

---

Theory is useful. Practice is essential. In this chapter, we build a complete test suite for a realistic web application. You will see the full BDD cycle in action: identify what to test, write the features, run them, and iterate.

---

## 4.1 Choosing What to Test First

When you start testing an existing application, the temptation is to test everything. Resist that temptation. Start small. Start with what matters most.

### The Testing Pyramid

<!-- diagram: test-pyramid -->

webship-js tests are at the top of the pyramid: end-to-end acceptance tests. They are slower than unit tests but they test the real user experience. Because they are slower, be strategic about what you test.

### What to Test First (Priority Order)

```
┌──────┬──────────────────────────────────────────────────────────┐
│ Rank │ What to Test                                             │
├──────┼──────────────────────────────────────────────────────────┤
│  1   │ Authentication (login, logout, session)                  │
│      │ → If users cannot log in, nothing else matters           │
├──────┼──────────────────────────────────────────────────────────┤
│  2   │ Critical business flows (checkout, signup, main feature) │
│      │ → The flows that generate revenue or provide core value  │
├──────┼──────────────────────────────────────────────────────────┤
│  3   │ Common user journeys (navigation, search, profile)       │
│      │ → The paths most users follow most of the time           │
├──────┼──────────────────────────────────────────────────────────┤
│  4   │ Admin functionality (user management, settings)          │
│      │ → Important but used less frequently                     │
├──────┼──────────────────────────────────────────────────────────┤
│  5   │ Edge cases and error handling                            │
│      │ → The things that break when you least expect it         │
├──────┼──────────────────────────────────────────────────────────┤
│  6   │ API endpoints                                            │
│      │ → Backend validation independent of UI                   │
└──────┴──────────────────────────────────────────────────────────┘
```

### Start with the Happy Path

For each feature, write the successful flow first. Then add failure scenarios. Then edge cases. This gives you the fastest coverage of the most important behaviors.

---

## 4.2 Building a Login Test Suite

Authentication is the gateway to your application. If login breaks, every authenticated feature breaks. Always test it first.

### Recipe 21: Testing Successful Login

**Problem:** A Friday evening deploy subtly broke the session cookie, and nobody could log in until Monday morning when the support tickets arrived in a pile. Every authenticated feature was down all weekend, and every authenticated test would have caught it -- if any had run. Login is the front door: when it breaks, the blast radius is the whole product. It deserves the first scenario you ever write for a project, and that scenario should run on every single deploy.

**Solution:**

Create `tests/features/authentication/login.feature`:

```gherkin
Feature: User Login
  As a registered user
  I want to log into my account
  So that I can access my personalized dashboard

  @smoke @auth @fast
  Scenario: Successful login with valid credentials
    Given I am on homepage
    When I go to "/login"
    Then I should see "Sign In"
    And I should see "Email"
    And I should see "Password"

    When I fill in "Email" with "testuser@example.com"
    And I fill in "Password" with "ValidPass123"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "Dashboard"
    And I should see "Welcome"
    And I should not see "Sign In"
```

**How It Works:**

This scenario tests the complete happy path:
1. Navigate to the login page
2. Verify the form exists (defensive check)
3. Fill in credentials
4. Submit the form
5. Verify the user lands on the dashboard
6. Verify the login form is no longer visible

The defensive check in the middle (`Then I should see "Sign In"`) catches a common problem: the test navigates to `/login` but the page did not load correctly. Without this check, the test would try to fill in fields that do not exist and give a confusing error. With it, a broken login PAGE fails at step three with "Sign In not found", while broken login LOGIC fails at the dashboard assertion -- two different bugs, immediately distinguishable in the report.

A few things happen between the lines. The fill steps resolve "Email" and "Password" label-first -- the visible label, then id, name, and placeholder -- so the scenario survives markup changes that keep the labels. After `I press "Sign In"`, webship-js's auto-settle hook is already waiting out the form submission; the explicit `wait for AJAX to finish` adds a larger budget and, just as importantly, documents in plain text that a server round-trip happens here. And the closing trio matters as a set: "Dashboard" proves navigation, "Welcome" proves the page rendered user content, and `I should not see "Sign In"` proves the app did not silently bounce back to the form -- the failure mode where login "works" but the session never sticks.

**At scale:** This scenario is deliberately shaped so it can be identical on every project you run -- only the labels and landing-page text vary. Make `login.feature` the first file in your starter template, tag it `@smoke`, and wire `--tags @smoke` into every deploy pipeline. When login coverage is a template inherited on day one rather than a task on a backlog, the weekend-outage story above stops being possible anywhere in the portfolio.

### Recipe 22: Testing Failed Login Attempts

**Problem:** The security review asks a simple question: "what does your login form do with bad input?" -- and the test suite has no answer, because everyone tested the door opening and nobody tested it staying shut. The expensive version of this gap is real: an application that said "wrong password" for existing accounts and "user not found" for everything else, quietly handing attackers a list of valid emails. Rejection behavior is a feature with its own requirements, and it needs its own scenarios.

**Solution:**

```gherkin
Feature: Login Failure Handling
  As a security-conscious application
  I want to reject invalid login attempts
  So that unauthorized users cannot access accounts

  Background:
    Given I am on homepage
    When I go to "/login"

  @auth @regression
  Scenario: Wrong password
    When I fill in "Email" with "testuser@example.com"
    And I fill in "Password" with "WrongPassword123"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "Invalid credentials"
    And I should not see "Dashboard"

  @auth @regression
  Scenario: Non-existent email
    When I fill in "Email" with "nobody@example.com"
    And I fill in "Password" with "AnyPassword123"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "Invalid credentials"
    And I should not see "Dashboard"

  @auth @regression
  Scenario: Empty email field
    When I fill in "Password" with "SomePassword"
    And I press "Sign In"
    Then I should see "Email is required"

  @auth @regression
  Scenario: Empty password field
    When I fill in "Email" with "testuser@example.com"
    And I press "Sign In"
    Then I should see "Password is required"

  @auth @regression
  Scenario: Both fields empty
    When I press "Sign In"
    Then I should see "Email is required"
```

**How It Works:**

Notice the Background section. Every scenario starts on the login page. We test five failure modes:
1. Correct email, wrong password
2. Non-existent email
3. Missing email
4. Missing password
5. Both missing

Also notice: the error message is generic ("Invalid credentials") for wrong password AND non-existent email. This is a security best practice -- do not reveal whether the email exists in your system. The two scenarios assert the SAME message on purpose; the day a developer "improves" the error to be more specific, this pair fails and forces the conversation about user enumeration before the change ships.

Look at which scenarios wait and which do not. The first two failure modes submit to the server, so they keep `wait for AJAX to finish`; the empty-field scenarios assert immediately, because required-field validation should fire client-side without a round-trip. That difference is itself part of the spec -- if "Email is required" suddenly needs an AJAX wait to pass, client-side validation broke and users are now paying a server round-trip to learn they forgot a field. The Background, meanwhile, is just the navigation; each scenario still runs in a fresh browser context, so a failed attempt in one scenario cannot trip a rate-limiter or leave error state for the next.

**At scale:** These five failure modes are a portfolio baseline, not a per-project invention. Put the five-scenario shape in your starter template next to the happy path from Recipe 21, and reviews become mechanical: which of the five is missing, and was that deliberate? When the security team audits 20 products, "every login form has the same five rejection scenarios, and the enumeration check is one of them" is an answer that ends the meeting early.

### Recipe 23: Testing Password Reset Flow

**Problem:** Password reset is the feature nobody exercises for months -- until a user is locked out, already frustrated, and discovers the reset link errors out because a refactor three releases ago broke a flow no test was watching. It is the classic silent-rot feature: rarely used, emotionally critical when used, and spanning a boundary (the email system) that makes people throw up their hands and not test it at all. You can test most of it today, and you should.

**Solution:**

```gherkin
Feature: Password Reset
  As a user who forgot their password
  I want to reset my password via email
  So that I can regain access to my account

  @auth @regression
  Scenario: Request password reset
    Given I am on homepage
    When I go to "/login"
    And I click "Forgot Password?"
    Then I should see "Reset Your Password"

    When I fill in "Email" with "testuser@example.com"
    And I press "Send Reset Link"
    And wait for AJAX to finish
    Then I should see "Check your email"
    And I should see "We have sent a password reset link"

  @auth @regression
  Scenario: Password reset with non-existent email
    Given I am on homepage
    When I go to "/login"
    And I click "Forgot Password?"
    And I fill in "Email" with "unknown@example.com"
    And I press "Send Reset Link"
    And wait for AJAX to finish
    # Security: same message whether email exists or not
    Then I should see "Check your email"

  @auth @regression
  Scenario: Password reset with invalid email format
    Given I am on homepage
    When I go to "/login"
    And I click "Forgot Password?"
    And I fill in "Email" with "not-an-email"
    And I press "Send Reset Link"
    Then I should see "Please enter a valid email"
```

**How It Works:**

Be precise about what these scenarios prove and what they do not. They verify the application's side of the contract: the link exists on the login page, the request form works, and the app responds with its promise ("Check your email"). They do NOT verify that an email actually arrived -- that requires pointing the test at a mail-catcher or checking via API, which Chapter 11's API steps make possible once you have one. Testing the reachable 80% today beats testing nothing while waiting for the perfect email harness.

The second scenario is the quiet star of this feature file. Same "Check your email" message for an address that does not exist -- the comment in the scenario says why, and the scenario enforces it. Reset forms are the OTHER classic user-enumeration leak (after login errors, Recipe 22): a form that answers "no account with that email" is a free account-discovery oracle. Note also that the invalid-format scenario asserts without an AJAX wait, expecting client-side validation, while the two server-bound scenarios wait -- the same client/server split you saw in Recipe 22, applied consistently.

### Recipe 24: Testing Session Persistence

**Problem:** The SSO migration shipped on Tuesday, and by Wednesday support was drowning: every page refresh logged users out, because the new token lived in JavaScript memory instead of a cookie. Login itself tested green -- you could sign in beautifully, once, until you touched F5. "Logged in" and "stays logged in" are different promises kept by different mechanisms, and a suite that only tests the first will wave the second's regressions straight through.

**Solution:**

```gherkin
Feature: Session Management
  As a logged-in user
  I want my session to persist as I navigate
  So that I do not need to log in on every page

  @auth @smoke
  Scenario: Session persists across pages
    Given I am a logged in user with the "admin" user
    When I go to "/dashboard"
    Then I should see "Dashboard"

    When I go to "/profile"
    Then I should see "Profile"
    And I should not see "Sign In"

    When I go to "/settings"
    Then I should see "Settings"
    And I should not see "Sign In"

  @auth @regression
  Scenario: Session persists after page reload
    Given I am a logged in user with the "admin" user
    When I go to "/dashboard"
    Then I should see "Dashboard"

    When I reload the page
    Then I should see "Dashboard"
    And I should not see "Sign In"
```

**How It Works:**

The two scenarios attack the two ways sessions die. Navigation between pages tests that the session credential travels with ordinary requests -- a cookie scoped to the wrong path, or a token attached only by one page's JavaScript, fails here. `When I reload the page` tests the harder case: a full reload tears down all in-memory JavaScript state, so anything not persisted in cookies or storage is gone. The in-memory-token bug from the problem statement passes scenario one and fails scenario two -- which is exactly why both exist. Within a scenario, webship-js keeps one browser context alive, so cookies and localStorage persist across these navigations just as they would for a real user; between scenarios, the context is rebuilt from scratch, so each scenario proves persistence independently rather than borrowing the previous one's session.

There is a second reason to keep this scenario green: webship-js's auth-state steps -- the "log in once, save the auth state, restore it everywhere" pattern -- work by snapshotting and re-injecting exactly this persisted state (cookies plus storage). If sessions do not survive a reload, saved auth state will not survive a restore either. This scenario is the canary for that whole optimization.

**At scale:** That makes this small feature file pull double duty across a portfolio. Once your projects adopt auth-state reuse to cut CI time (and they should -- it is usually the biggest runtime win available, see Recipe 4's numbers), every suite silently depends on session persistence working. Keep this scenario tagged `@smoke` on every project: when an SSO or cookie-policy change breaks persistence, you want the named, two-scenario canary failing loudly -- not forty unrelated scenarios failing mysteriously because their restored sessions evaporated.

---

## 4.3 Building a Registration Test Suite

### Recipe 25: Testing Complete Registration

**Problem:** Marketing launches the spring campaign on Monday, traffic triples, and Tuesday's standup learns that the redesigned signup page has been silently rejecting everyone whose browser autofilled the Terms checkbox differently. Registration is the revenue front door -- every broken hour is measurable in lost accounts -- and yet it is usually tested less than login, because "we would notice". You notice faster with a scenario that walks the whole form on every deploy.

**Solution:**

```gherkin
Feature: User Registration
  As a new visitor
  I want to create an account
  So that I can access member features

  @auth @smoke
  Scenario: Successful registration
    Given I am on homepage
    When I go to "/register"
    And I fill in "First Name" with "Test"
    And I fill in "Last Name" with "User"
    And I fill in "Email" with "newuser-test@example.com"
    And I fill in "Password" with "SecurePass123!"
    And I fill in "Confirm Password" with "SecurePass123!"
    And I check "I agree to the Terms of Service"
    And I press "Create Account"
    And wait for AJAX to finish
    Then I should see "Welcome"
    And I should see "Your account has been created"
```

**How It Works:**

Every input type on the form gets exercised through its label. The text fields go through the usual label-first resolution, and `I check "I agree to the Terms of Service"` does the same for the checkbox -- it finds the checkbox by its visible label text and checks it, which matters because terms checkboxes are exactly the elements designers love to restyle into custom widgets where the real `<input>` is hidden. If a redesign breaks the label-to-input association (the autofill bug in the problem statement was one of these), this step fails with a clear "cannot find" error instead of a silent un-checked box. The final two assertions are deliberately both present: "Welcome" alone could be the homepage greeting; "Your account has been created" pins the success to THIS action.

The trap in this recipe is the email address. `newuser-test@example.com` registers fine on the first run -- and on the second run the same address collides with the account the first run created, and your registration test fails for being too successful. Decide on day one how the suite gets a fresh identity each run: a test-data reset job before the suite, an environment that wipes between nightly runs, or a custom step that generates a unique address (Chapter 17 territory). Any of the three works; having none of them means a registration suite that only passes on Mondays.

**At scale:** Test-data hygiene is a portfolio policy, not a per-suite hack. Pick ONE of the three strategies above and make it the standard -- mixed strategies mean the person debugging project N first has to discover which trick project N uses. The reset-before-run approach is usually the right default for staging environments because it also serves Recipe 27, which needs a known account to already exist: one seeding script can guarantee both "this email is free" and "that email is taken".

### Recipe 26: Testing Field Validation with Scenario Outline

**Problem:** The registration form's validation rules live in a spreadsheet the backend developer keeps locally, and the only executable record of them is scattered through nine copy-pasted scenarios that differ by two words each. When the password policy changes from 8 characters to 10, someone updates the application, the spreadsheet -- and five of the nine scenarios. The other four keep testing the old rule, passing for the wrong reasons. Validation rules are tabular by nature; the tests should be tabular too, with exactly one place to edit.

**Solution:**

```gherkin
Feature: Registration Validation
  As the application
  I want to validate registration fields
  So that only valid data enters the system

  Scenario Outline: Field validation rules
    Given I am on homepage
    When I go to "/register"
    And I fill in "<field>" with "<value>"
    And I press "Create Account"
    Then I should see "<error>"

    Examples: Email validation
      | field  | value             | error                         |
      | Email  | not-an-email      | Please enter a valid email    |
      | Email  | @missing-local    | Please enter a valid email    |
      | Email  | spaces in@mail.co | Please enter a valid email    |

    Examples: Password validation
      | field    | value    | error                              |
      | Password | 123      | Password must be at least 8 chars  |
      | Password | password | Password must include a number     |
      | Password | 12345678 | Password must include a letter     |

    Examples: Name validation
      | field      | value | error                    |
      | First Name |       | First name is required   |
      | Last Name  |       | Last name is required    |
```

This generates **9 test scenarios** from a single template, covering email, password, and name validation rules.

**How It Works:**

Each Examples row becomes a complete, independent scenario -- fresh browser context, fresh navigation to `/register`, one field filled, one submit, one assertion. The labeled Examples blocks are not decoration: when the run reports a failure, it names the block and the row, so "Password validation, row `| password | Password must include a number |`" tells you which rule regressed before you open anything. Note there is no `wait for AJAX to finish` anywhere in the template -- these rules should fire client-side, and the outline quietly enforces that expectation for all nine cases at once.

Mind the cost model: nine rows means nine browser sessions, which is the right price for nine real business rules. What the outline must not become is a fuzzer -- if you want to throw 200 malformed emails at the validator, do that in a unit test against the validation function, and keep the acceptance rows to the cases a product owner would recognize as policy.

**At scale:** This table IS the validation spec, in a format a product owner can edit. When the password policy changes, the policy change arrives as a PR that edits one row -- visible, reviewable, and impossible to half-apply the way the nine scattered scenarios were. Share the outline's shape across projects and the spreadsheet-on-someone's-laptop problem disappears portfolio-wide: the rules live in the repo, and the nightly run notarizes them.

### Recipe 27: Testing Duplicate Email Prevention

**Problem:** Support finds a customer with three accounts under one email -- three carts, three loyalty balances, and a password reset that updates a different account than the one she logs into. The uniqueness check existed once, then an auth refactor moved account creation to a new endpoint and the check did not move with it. No single feature "looked" broken, which is why this regression class ships so easily: only a test that tries the duplicate, on every deploy, keeps the constraint honest.

**Solution:**

```gherkin
Feature: Duplicate Email Prevention
  As the application
  I want to prevent duplicate registrations
  So that each email has only one account

  @auth @regression
  Scenario: Registration with already-used email
    Given I am on homepage
    When I go to "/register"
    And I fill in "First Name" with "Another"
    And I fill in "Last Name" with "User"
    And I fill in "Email" with "existing@example.com"
    And I fill in "Password" with "SecurePass123!"
    And I fill in "Confirm Password" with "SecurePass123!"
    And I check "I agree to the Terms of Service"
    And I press "Create Account"
    And wait for AJAX to finish
    Then I should see "An account with this email already exists"
    And I should not see "Welcome"
```

**How It Works:**

This scenario has a hidden dependency, and it is healthier to say so out loud: `existing@example.com` must already have an account in the environment under test, or the registration succeeds and the scenario fails for the wrong reason. That makes this a fixture-dependent test. The clean way to satisfy it is the seeding script from Recipe 25's At-scale note -- the same job that frees up the fresh address guarantees the taken one. The self-contained alternative is a scenario that registers an email and then immediately tries to register it again; it removes the fixture dependency at the cost of a longer scenario and a per-run unique address. Either is fine. What is not fine is the implicit version, where the test passes only because someone manually created that account on staging in 2024.

The closing pair of assertions mirrors the constraint pattern from Chapter 3: the positive ("already exists" message appeared) proves the application responded as designed, and the negative ("Welcome" absent) proves no second account slipped through behind a friendly-looking error. One last design question worth a Three Amigos minute: should THIS error be specific while login errors stay generic? Most teams accept the enumeration trade-off here -- a registration form necessarily reveals taken emails -- but it should be a recorded decision, not an accident.

**At scale:** Fixture data is an environment contract, and it belongs in version control next to the suite -- a seed script or documented fixture list that says "these accounts exist, with these roles, on every test environment". When 20 projects each assume slightly different invisible staging state, environment rebuilds become archaeology. When the assumptions are a checked-in script, rebuilding staging for any project is one command, and this scenario passes on a fresh environment the same day.

---

## 4.4 The Red-Green-Refactor Cycle in Practice

Let us walk through the full BDD cycle for a single scenario.

### Step 1: Write the Scenario (RED)

```gherkin
Feature: User Profile
  Scenario: User views their profile
    Given I am a logged in user with the "admin" user
    When I go to "/profile"
    Then I should see "My Profile"
    And I should see "admin"
```

### Step 2: Run It

```bash
npm test
```

Output (if the feature does not exist yet):

```
1 scenario (1 failed)
4 steps (2 passed, 1 failed, 1 skipped)

Failing step:
  When I go to "/profile"
  Error: Navigation to http://localhost:8080/profile failed: 404
```

This is RED. The `/profile` page does not exist yet. That is expected and correct.

### Step 3: Implement (GREEN)

Build the `/profile` page in your application. Show the user's name. Then run the test again:

```bash
npm test
```

```
1 scenario (1 passed)
4 steps (4 passed)
```

GREEN. The scenario passes.

### Step 4: Refactor

Now improve the code. Maybe extract a profile component, clean up the routing, improve the styling. Run the tests after each change:

```bash
npm test
```

If the tests still pass, your refactoring did not break anything. If a test fails, your refactoring changed behavior -- fix it.

### The Safety Net

This is the power of BDD. Every time you refactor, the test suite tells you immediately if you broke something. You can refactor with confidence because you have a safety net of executable specifications.

---

## 4.5 Running and Interpreting Results

### Recipe 28: Understanding Cucumber Output States

**Problem:** A teammate pastes a wall of dots and one red `F` into the team chat with "tests are broken, help." Before anyone can help, they have to know what the symbols mean -- and the difference between a `F`, a `U`, and an `A` is the difference between a real bug, a step you forgot to write, and two step definitions fighting over the same sentence. Until the whole team reads these six states the same way, every failure turns into a guessing game.

**Solution:**

```
┌─────────────┬────────┬────────────────────────────────────────────┐
│ State       │ Symbol │ What It Means                              │
├─────────────┼────────┼────────────────────────────────────────────┤
│ Passed      │   .    │ Step executed successfully                  │
│             │ (green)│ The assertion was met                      │
├─────────────┼────────┼────────────────────────────────────────────┤
│ Failed      │   F    │ Step executed but assertion failed          │
│             │ (red)  │ OR an error/exception occurred             │
│             │        │ Remaining steps in scenario are SKIPPED    │
├─────────────┼────────┼────────────────────────────────────────────┤
│ Undefined   │   U    │ No step definition matched this step text  │
│             │(yellow)│ Cucumber will suggest a snippet to use     │
├─────────────┼────────┼────────────────────────────────────────────┤
│ Pending     │   P    │ Step definition exists but is incomplete   │
│             │(yellow)│ Contains 'return "pending"' or similar     │
├─────────────┼────────┼────────────────────────────────────────────┤
│ Skipped     │   -    │ Not executed because a previous step failed│
│             │ (cyan) │ System in unknown state, unsafe to continue│
├─────────────┼────────┼────────────────────────────────────────────┤
│ Ambiguous   │   A    │ Multiple step definitions match this text  │
│             │ (red)  │ Fix: make step patterns more specific      │
└─────────────┴────────┴────────────────────────────────────────────┘
```

**How It Works:**

The two states people confuse most are Failed and Undefined, and they point at opposite problems. A red `F` means the step ran and the application did the wrong thing -- a genuine signal worth waking someone for. A yellow `U` means Cucumber never found a step definition matching that sentence, so nothing about the application was tested at all; the suite is reporting on a gap in your test code, not a gap in the product. Treating a `U` like an `F` sends a developer hunting through application code for a bug that lives in the feature file.

Skipped (`-`) is the one that protects you. The moment a step fails, Cucumber abandons the rest of that scenario, because once an action did not happen the page is in an unknown state and every later assertion would be lying. So a scenario reports one real failure followed by a trail of cyan skips -- the skips are not extra problems, they are Cucumber refusing to guess. Ambiguous (`A`) almost always means two people wrote a step definition for the same phrase in different files; with webship-js loading every file under `tests/step-definitions/**/*.js` into one global registry, the fix is to make one pattern more specific or delete the duplicate.

**At scale:** When twenty projects share one chat channel, agree on what each state obligates. A red `F` pages the on-call engineer; a `U` or `A` is a test-code defect that goes to whoever wrote the feature, never to the product team; a wall of skips after one failure is one bug, not twenty. Writing that convention down once -- and pointing every new hire at this table -- stops the daily "is this real?" triage tax that otherwise scales linearly with the number of suites you run.

### Recipe 29: Using Different Reporters

**Problem:** The format that helps a developer watch a local run -- one line per step, scrolling past -- is useless to a CI server that just needs a compact pass/fail stream, and worse than useless to a release manager who wants a link they can open in a browser. One run, three audiences, and if you only know one reporter you end up serving all three badly.

**Solution:**

```bash
# Progress bar (default - great for CI)
npx cucumber-js --format progress-bar

# Detailed output (shows each step)
npx cucumber-js --format progress

# Summary only (just pass/fail counts)
npx cucumber-js --format summary

# JSON output (for reporting tools)
npx cucumber-js --format json:reports/results.json

# HTML report (visual, shareable)
npx cucumber-js --format html:reports/report.html

# Multiple formats at once
npx cucumber-js \
  --format progress-bar \
  --format json:reports/results.json \
  --format html:reports/report.html
```

**How It Works:**

Reporters are independent consumers of the same run -- you are not re-running tests to get a second format, you are attaching another listener to the one execution. That is why you can stack `--format` flags: the progress bar prints to your terminal for a human watching, while `json:reports/results.json` writes the machine-readable record to disk in the same pass. The JSON file is the important one for teams. It is the canonical artifact webship-js's own report generator (`bin/generate-reports.js`) reads to build the HTML and PDF, and it carries every step, status, timing, and attached screenshot, so it is the single source everything downstream is derived from.

The default `progress-bar` is deliberately terse because CI logs are read by machines and skimmed by humans -- one character per step keeps a thousand-step run to a few screens. The verbose `progress` format is for the moment you are debugging a specific scenario locally and want to see each step's text scroll by.

**At scale:** Standardize on writing the JSON report at a fixed path in every project, even the ones nobody has wired a dashboard to yet. The terminal format is a personal preference that can vary by developer; the JSON artifact is a contract. Once every suite across the portfolio emits `reports/results.json` in the same place, one CI template can collect them, one script can roll them into a cross-project quality dashboard, and a new project joins the dashboard for free the day it lands -- no per-project reporting code to write.

### Recipe 30: Generating HTML Reports

**Problem:** Your suite is green, but the delivery manager three time zones away cannot run `npm test` and would not read terminal output if they could. They keep asking "is the release safe?" in standups, and "trust me, the tests pass" stops being a satisfying answer around the third release. They need something they can open in a browser and read themselves.

**Solution:**

Add HTML reporting to your `cucumber.js`:

```javascript
module.exports = {
  default: {
    require: ['node_modules/webship-js/tests/step-definitions/*.js'],
    paths: ['tests/features/**/*.feature'],
    worldParameters: { launchUrl: 'http://localhost:8080' },
    format: [
      'progress-bar',
      'html:reports/cucumber-report.html'
    ],
    publishQuiet: true
  }
};
```

Run tests:

```bash
npm test
```

Open the report:

```bash
open reports/cucumber-report.html
# or on Linux:
xdg-open reports/cucumber-report.html
```

The HTML report shows:
- Total scenarios passed/failed
- Each feature with its scenarios
- Each step with pass/fail status
- Error messages and stack traces for failures
- Execution time per scenario

**How It Works:**

The built-in `html` formatter is the quick option -- Cucumber writes a self-contained page as the run finishes, no extra step. For a report you actually hand to a stakeholder, webship-js ships `bin/generate-reports.js`, which reads the JSON artifact from Recipe 29 and produces a richer HTML or PDF document with selectable themes and a metadata header. Crucially, the screenshots webship-js captures on failure (it sets `WEBSHIP_SCREENSHOT_ON_FAILED` true by default) are embedded right into that report, so a failure shows the picture of the broken page next to the error -- a non-engineer can see what went wrong without reading a stack trace. That turns the report from a pass/fail log into evidence.

**At scale:** A green checkmark in CI tells a manager the run finished; an HTML or PDF report tells them *what was verified*. Stamp each report with `--app-version`, `--env`, and `--browser` (the metadata flags `generate-reports.js` accepts) so a release manager comparing two builds, or an auditor six months later asking "what did we test for the March release?", reads the answer straight off the document. When every project emits the same report shape, the question stops being "do you trust QA?" and becomes "open the report" -- which is exactly the cross-project, manager-readable quality signal a PMO needs and the kind of self-serve evidence that ends the standup interrogation.

---

## 4.6 Your First Test Suite Checklist

Before you call your initial test suite "done," make sure you have covered:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TEST SUITE CHECKLIST                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  □  Project Setup                                               │
│     □  package.json with test scripts                           │
│     □  cucumber.js with correct launchUrl                       │
│     □  Directory structure (features/, step-definitions/, etc)  │
│     □  .gitignore (node_modules/, reports/)                     │
│                                                                 │
│  □  Smoke Tests (run on every deployment)                       │
│     □  Homepage loads                                           │
│     □  Login works                                              │
│     □  Main navigation works                                    │
│     □  Critical feature accessible                              │
│                                                                 │
│  □  Authentication Tests                                        │
│     □  Successful login                                         │
│     □  Failed login (wrong password)                            │
│     □  Failed login (non-existent user)                         │
│     □  Empty field validation                                   │
│     □  Logout                                                   │
│     □  Session persistence                                      │
│                                                                 │
│  □  Core Feature Tests                                          │
│     □  Happy path for the main feature                          │
│     □  At least one failure scenario                            │
│     □  At least one edge case                                   │
│                                                                 │
│  □  Organization                                                │
│     □  Features organized by domain                             │
│     □  Tags applied (@smoke, @regression, @auth, etc.)         │
│     □  Background used for shared setup                         │
│     □  Scenario Outline used for data-driven tests              │
│                                                                 │
│  □  CI Integration                                              │
│     □  Tests run in CI pipeline                                 │
│     □  Reports generated                                        │
│     □  Build fails on test failure                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## A Real-World Example: Blog Application Test Suite

Let me show you what a complete test suite looks like for a blog application. This ties together everything from Chapters 1-4.

### File Structure

```
blog-tests/
├── cucumber.js
├── package.json
├── tests/
│   ├── features/
│   │   ├── smoke/
│   │   │   └── health-check.feature
│   │   ├── authentication/
│   │   │   ├── login.feature
│   │   │   └── registration.feature
│   │   ├── blog/
│   │   │   ├── create-post.feature
│   │   │   ├── edit-post.feature
│   │   │   ├── view-posts.feature
│   │   │   └── comments.feature
│   │   └── admin/
│   │       └── user-management.feature
│   └── assets/
│       └── test-image.png
└── reports/
```

### health-check.feature

```gherkin
@smoke @fast
Feature: Health Check
  As a DevOps engineer
  I want to verify the site is up
  So that I know deployments succeeded

  Scenario: Homepage loads
    Given I am on homepage
    Then I should see "Blog"

  Scenario: Login page loads
    Given I am on homepage
    When I go to "/login"
    Then I should see "Sign In"
```

### create-post.feature

```gherkin
@javascript @blog
Feature: Create Blog Post
  As a blog author
  I want to create new blog posts
  So that I can share my thoughts with readers

  Background:
    Given I am a logged in user with the "editor" user
    And I go to "/admin/posts/new"
    And wait for AJAX to finish

  @smoke
  Scenario: Create a simple blog post
    When I fill in "Title" with "Getting Started with BDD"
    And I fill the rich text editor field "Body" with "<p>BDD is a powerful approach to software development.</p>"
    And I press "Publish"
    And wait for AJAX to finish
    Then I should see "Post published successfully"

  @regression
  Scenario: Create post with all fields
    When I fill in "Title" with "Complete Guide to webship-js"
    And I fill the rich text editor field "Body" with "<p>Learn automated testing with webship-js.</p>"
    And I fill in "Excerpt" with "A comprehensive guide"
    And I select "Technology" from "Category"
    And I fill in "Tags" with "bdd, testing, automation"
    And I attach the file "test-image.png" to "Featured Image"
    And I press "Publish"
    And wait for AJAX to finish
    Then I should see "Post published successfully"

  @regression
  Scenario: Cannot create post without title
    And I fill the rich text editor field "Body" with "<p>Some content</p>"
    And I press "Publish"
    Then I should see "Title is required"

  @regression
  Scenario: Create post as draft
    When I fill in "Title" with "Draft Post"
    And I fill the rich text editor field "Body" with "<p>Work in progress.</p>"
    And I press "Save as Draft"
    And wait for AJAX to finish
    Then I should see "Draft saved"
```

### Running the Suite

```bash
# Run everything
npm test

# Run just smoke tests
npm run test:smoke

# Run just blog tests
npx cucumber-js --tags "@blog"

# Run blog tests excluding WIP
npx cucumber-js --tags "@blog and not @wip"
```

---

## Chapter Summary

- Start testing with authentication (login/logout), then critical business flows
- Follow the testing pyramid: E2E tests (webship-js) at the top, focused on features
- Begin with the happy path, then add failure and edge case scenarios
- The Red-Green-Refactor cycle works at the feature level: write scenario → fail → implement → pass → refactor
- Use reporters (progress-bar, HTML, JSON) for different audiences
- A production-ready test suite has smoke tests, auth tests, core feature tests, and CI integration
- Organize features by domain, use tags for filtering, Background for shared setup

Part I is complete. You now understand BDD, have webship-js set up, know Gherkin inside out, and have built a real test suite. In Part II, we dive into the recipes -- specific solutions for every type of web testing challenge.

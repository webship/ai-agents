# Chapter 23: Test Maintenance

---

Writing tests is the beginning. Maintaining them is the real work. Applications evolve: new features land, old features change, UIs get redesigned, APIs get versioned. Every change potentially affects your test suite. Without a maintenance discipline, the suite rots until it is more expensive to maintain than the bugs it catches.

And maintenance debt compounds across projects. A team that keeps one suite healthy by heroics will not keep twenty healthy the same way -- the redesign that costs you an afternoon on one site costs a month across a portfolio, unless the suites share conventions, shared steps, and a review discipline. Every recipe in this chapter is written with that multiplication in mind.

---

## Recipe 211: Refactoring Steps When the UI Changes

**Problem:** Monday morning, the redesigned login page ships. The "Sign In" button is now "Log In," the "Email" field is now "Username," and every suite that logs in -- which is every suite -- goes red at once. The team's first instinct is to panic; yours should be to notice that this is the *good* failure mode: the tests are telling you exactly what changed, in the language users see.

**Solution:**

```gherkin
# Before the UI change:
When I fill in "Email" with "admin@example.com"
And I press "Sign In"

# After the UI change:
When I fill in "Username" with "admin@example.com"
And I press "Log In"
```

**Approach:**
1. Run the test suite -- identify which tests fail
2. Update Gherkin steps to match the new UI labels
3. If you have custom step definitions with hardcoded selectors, update those
4. Run the suite again to verify all fixes

**How It Works:** the reason this refactor is a find-and-replace instead of an archaeology dig is webship-js's selector strategy. `I fill in "Email"` resolves label-first -- label text, then id, name, placeholder -- and `I press "Sign In"` finds the button by its accessible name through `getByRole`. The test's contract is the words on the screen, so when the words change, the *Gherkin* changes -- a diff any product owner can read and approve -- while the step definitions stay untouched. Compare that with a selector-based suite, where the same redesign means hunting CSS classes through support code. The depth of the change matches the depth of what changed: copy change, copy-level fix.

**At scale:** centralize before the redesign, not after. If login lives in a `Background:` or one shared step per project, the rename is one edit per repo; if forty feature files each spell out the login form, it is forty edits times twenty projects. Where you do need raw selectors, keep them in named-selector JSON files under `tests/selectors/` so a markup change is one JSON edit. The redesign that hits a prepared portfolio is an afternoon; the same redesign on an unprepared one is the sprint nobody planned for.

---

## Recipe 212: Updating Tests for New Features

**Problem:** Security ships two-factor authentication on a Thursday. It is a genuine improvement -- and it just inserted a step into the middle of every logged-in scenario you own. Two hundred scenarios across a dozen features now stall at a verification prompt none of them know exists. The wrong fix is editing two hundred scenarios; the right fix is editing the one step they all share.

**Solution:**

```javascript
// Add a new step for the 2FA code
When('I enter the 2FA code', async function () {
  // In test environment, the 2FA code is always '123456'
  await this.page.getByLabel('Verification Code').fill('123456');
  await this.page.getByRole('button', { name: 'Verify' }).click();
  await this.page.waitForLoadState('networkidle');
});
```

Update the login step that many tests depend on:

```javascript
Given('I am a logged in user with the {string} user', async function (role) {
  const user = users[role];
  await this.page.goto(this.launchUrl + '/login');
  await this.page.waitForLoadState('networkidle');
  await this.page.getByLabel('Email').fill(user.email);
  await this.page.getByLabel('Password').fill(user.password);
  await this.page.getByRole('button', { name: 'Sign In' }).click();
  await this.page.waitForLoadState('networkidle');

  // Handle 2FA if present
  const twoFaInput = this.page.getByLabel('Verification Code');
  if (await twoFaInput.isVisible({ timeout: 2000 }).catch(() => false)) {
    await twoFaInput.fill('123456');
    await this.page.getByRole('button', { name: 'Verify' }).click();
    await this.page.waitForLoadState('networkidle');
  }
});
```

Now all tests that use `Given I am a logged in user` continue to work without changes.

**How It Works:** this is the payoff of funneling shared behavior through one step definition: the 2FA handling lands in one place, behind the same Gherkin sentence two hundred scenarios already speak. The conditional check (`isVisible` with a short timeout, swallowing the miss) makes the step tolerant of both worlds -- environments where 2FA is enabled and environments where it is not yet -- which matters mid-rollout when staging has the feature and the demo site does not. The hardcoded `123456` is the standard trick of pinning the verification code in the test environment; the alternative, scraping real codes from email or TOTP, buys realism you rarely need at the cost of flake you definitely do not.

**At scale:** logging in through the form in every scenario is also a speed and stability tax that grows with feature complexity -- 2FA just made every scenario two steps slower. webship-js ships auth-state steps that save and restore the browser's `storageState`: log in once, save the state, and let subsequent scenarios restore it instead of re-running the form. Across a portfolio, pair that with one rule: every project uses the same login step sentence. Then a platform-wide auth change (SSO migration, CAPTCHA, new 2FA flow) is one step-definition edit per project -- or one edit in a shared step package -- instead of an all-hands week.

---

## Recipe 213: Removing Obsolete Tests

**Problem:** The wishlist feature was retired two quarters ago, but its scenarios still run nightly -- failing, then skipped, then tagged `@flaky`, then quietly ignored. Now they are noise in every report, minutes in every CI run, and a small lie in your coverage numbers. Dead tests are worse than no tests: they train the team to ignore red.

**Checklist:**

```
Before removing a test, verify:
  □ The feature was intentionally removed (not just temporarily hidden)
  □ The test is not covering behavior that still exists elsewhere
  □ The team agrees the test should be removed (PR review)
  □ The feature file deletion is in its own commit (easy to revert)
```

**Process:**
1. Run the test -- if it fails with "page not found" or "element not found," the feature is gone
2. Check with the product owner: "Is this feature permanently removed?"
3. Delete the feature file
4. Delete any custom step definitions that were ONLY used by that feature
5. Run the remaining test suite to ensure nothing else depended on the deleted steps

**How It Works:** the product-owner check in step 2 is the load-bearing one -- "hidden behind a feature flag for one client" and "removed" look identical to a failing test, and only the PO knows which it is. This is a place where readable Gherkin earns its keep: you can paste the scenario into the question and get a real answer. Step 5 is cheap insurance done right: after deleting steps, run `npx cucumber-js --dry-run` -- it verifies in seconds that every remaining step still has a definition, without burning a full browser run. Keeping the deletion in its own commit means that when the feature comes back next year (features do), `git revert` resurrects the tests intact.

**At scale:** schedule pruning instead of waiting for it to hurt -- a quarterly pass per project, looking for scenarios that have not changed status in months and tags that no pipeline selects anymore. Track scenario count alongside pass rate in your cross-project reporting: a suite that only ever grows is not a healthier suite, it is a slower one. Deleting a dead test is a maintenance win worth the same review ceremony as adding a live one.

---

## Recipe 214: Managing Test Data Over Time

**Problem:** The suite was born against a database where "testuser1" existed and the catalog had exactly twelve products. A year later, a schema migration renamed a column, someone cleaned up staging, and a marketing import added four hundred products. None of your code changed -- but a third of the suite fails, and the new QA engineer spends her first week discovering that the tests do not document what data they expect. Stale data is the slowest-burning form of maintenance debt because nothing announces it.

**Solution:**

```
Test Data Management Strategies:
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Strategy 1: Create data per test (recommended)                  │
│  Each test creates what it needs via API or UI                   │
│  Pros: Independent, no stale data, runs anywhere                 │
│  Cons: Slower setup per test                                     │
│                                                                  │
│  Strategy 2: Seed script before suite                            │
│  A script runs before all tests to set up known state            │
│  Pros: Fast test execution, known state                          │
│  Cons: Seed script must be maintained, can get out of sync       │
│                                                                  │
│  Strategy 3: Database snapshot/restore                           │
│  Snapshot a clean database, restore before each test run         │
│  Pros: Fastest, always consistent                                │
│  Cons: Snapshot must be updated when schema changes              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**How It Works:** the strategies differ in *where the truth about test data lives*. With per-test creation, the truth is in the scenario itself -- `Given there is a user named "Sara"` both documents and provisions, usually through the API steps (`When I send a POST request to "/api/users" with values:`), which cost milliseconds instead of UI seconds. With a seed script, the truth lives in one versioned script that drifts unless someone owns it. With snapshots, the truth is a binary blob nobody can read in review. That is why the recommendation is per-test creation by default, with seeds or snapshots reserved for the expensive substrate -- roles, taxonomies, configuration -- that every scenario shares and no scenario should rebuild. Mixed mode is normal: seed the skeleton, create the flesh per scenario.

**At scale:** per-test creation is also what makes one suite pattern portable across twenty projects, because it removes "prepare the database" from every onboarding checklist -- point `LAUNCH_URL` at any environment and run. This was the working rule for testing the Varbase distribution across many sites: the installation profile provides the same baseline everywhere, and every scenario builds its own content on top. If a scenario only passes on one specially-groomed environment, treat that as a defect in the scenario.

---

## Recipe 215: Code Review Checklist for Tests

**Problem:** Application code gets two reviewers and a style bot; test code gets "LGTM, it's green". Then the new hire's first PR lands with a `wait 5s`, a hardcoded staging URL, and a step definition that duplicates one three files away -- and it merges, because nobody knew what to look for. Six months of that is how a suite rots while every individual PR looked fine.

When reviewing a PR that includes test changes:

```
Test PR Review Checklist:
  □ Scenarios describe behavior, not implementation
  □ Step text uses business language
  □ No hardcoded wait times where smart waits would work
  □ No shared state between scenarios
  □ Tests create their own data (no fixture dependencies)
  □ Assertions check both positive and negative cases
  □ Tags are consistent with project conventions
  □ New step definitions do not duplicate existing ones
  □ Error messages in custom assertions are descriptive
  □ No console.log or debug statements left in
```

**How It Works:** every line of the checklist encodes one of the failure modes from Chapter 21, caught at the cheapest possible moment -- before merge. "No hardcoded wait times" blocks the static sleeps that smartSettle exists to replace. "Tests create their own data" blocks the fixture coupling that breaks the third environment you deploy to. "New step definitions do not duplicate existing ones" is the big one on a mature project: with 411 built-in steps plus your custom ones, the odds are good the step already exists -- a quick `--dry-run`, or a grep through `tests/step-definitions/`, beats writing a near-twin that drifts apart from the original. The repo's own rule is worth adopting too: every new step definition gets documented with at least five example lines, so the next reviewer can tell at a glance what it is for.

**At scale:** make the checklist a PR template in every repository -- the same items, the same order, across all twenty projects. That does two things: review quality stops depending on which senior happens to be on rotation, and engineers who move between projects already know the bar. For a QA lead, the checklist is also a teaching document -- the new hire's first PR gets reviewed against written criteria instead of folklore, and the feedback points at a line in the template, not at a person's opinion.

---

## Chapter Summary

- Test maintenance is an ongoing discipline, not a one-time task -- and it compounds across projects
- UI label changes require Gherkin updates, not step definition changes -- label-first and role-based selectors make it so
- New features may require updating shared steps (like login); shared steps are why that is cheap
- Remove obsolete tests deliberately, not accidentally -- and verify with `--dry-run` after deleting steps
- Manage test data through per-test creation by default; seed only the shared substrate
- Use the same code review checklist for test PRs across every project

Next: Performance Optimization -- making your test suite faster.

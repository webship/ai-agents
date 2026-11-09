# Chapter 26: Testing CMS Applications

---

Content Management Systems (WordPress, Drupal, Joomla, headless CMS platforms) have unique testing patterns. Content workflows involve creating drafts, editing, publishing, unpublishing, and managing media. User roles in a CMS are more granular than in a typical app -- content creators, editors, reviewers, publishers, and administrators each have different permissions.

CMS testing is also where the one-team-many-projects reality bites hardest. This is the world I come from: at Vardot we maintained the Varbase distribution, and one suite pattern had to hold across dozens of Drupal sites that shared the same editorial workflows but each carried its own theme, modules, and content model. The recipes in this chapter are shaped by that experience -- they lean on labels and roles instead of markup, because labels and roles are what survive a CMS upgrade.

This chapter provides recipes tailored to CMS testing patterns.

---

## Recipe 225: Testing Content Creation Workflow

**Problem:** A CMS earns its keep the moment an editor writes something, which makes content creation the flow you can least afford to break. The moment it hurts is the Tuesday you roll out a core or plugin update: a JavaScript regression in the editor quietly breaks "Save as Draft", and by lunchtime every editor on the client's team is blocked -- and their support ticket reaches you before your own monitoring does. You want the editor's first job of the day, write an article and save a draft, proven on every build instead of discovered by the client.

**Solution:**

```gherkin
Feature: Content Creation
  As a content editor
  I want to create articles
  So that I can publish content on the website

  Scenario: Create a new article
    Given I am a logged in user with the "editor" user
    When I go to "/admin/content/add"
    And wait for AJAX to finish
    And I fill in "Title" with "Getting Started with BDD"
    And I fill in "Body" with "Behaviour-Driven Development is a practice..."
    And I select "Technology" from "Category"
    And I press "Save as Draft"
    And wait for AJAX to finish
    Then I should see "Content saved"
    And I should see "Status: Draft"

  Scenario: Create article with rich text
    Given I am a logged in user with the "editor" user
    When I go to "/admin/content/add"
    And wait for AJAX to finish
    And I fill in "Title" with "Formatted Article"
    And I fill in the rich text editor "Body" with "Bold and italic content"
    And I press "Save as Draft"
    And wait for AJAX to finish
    Then I should see "Content saved"
```

**How It Works:** `Given I am a logged in user with the "editor" user` performs a real login as the named test user, so every step after it runs with editor permissions -- the same lens the real editor sees through. The fill steps do more than they look like: `I fill in "Title" with ...` resolves the field label-first, falling back to id, name, and placeholder only when no label matches. That ordering is the survival trait for CMS tests, because admin themes love to rewrite ids and class names between versions while the human-visible label "Title" stays put. The rich-text step exists because WYSIWYG editors (CKEditor, TinyMCE) replace the textarea with their own editing surface -- typing into the hidden textarea would do nothing, so the dedicated step targets the editor itself. The `wait for AJAX to finish` lines hold the scenario while the admin page does its constant background chatter -- autosave, editor initialization, autocomplete lookups -- by watching `window.__webshipAjaxCount`, the fetch/XHR counter webship-js injects into every page, until in-flight requests reach zero. And after every state-changing step, webship-js runs its smartSettle check automatically (requests at zero, no pending timers, DOM quiet for 250ms), so the explicit waits are belt and braces around the heaviest moments, not the only thing holding back a race.

**At scale:** If your team runs one platform across many sites -- one Drupal distribution on thirty client projects, say -- this feature can be literally identical everywhere, with only `LAUNCH_URL` changing per site. That is how the Varbase suite worked: one set of editorial features, one environment variable per site, and a fleet-wide run after every release candidate. The discipline that makes it possible: keep role names and test-user fixtures identical across projects ("editor" means the same thing on every site), and never assert on theme-specific markup.

---

## Recipe 226: Testing Content Publishing

**Problem:** Everyone tests their own step of the publishing workflow and nobody tests the handoffs. The editor checks that drafts save, the admin checks that the publish button works -- and then a permissions tweak breaks "Submit for Review", content piles up in the pending queue for two days, and the marketing lead asks why the product launch post never went live. Workflow bugs live between roles, so the test has to cross roles too.

**Solution:**

```gherkin
Feature: Content Publishing Workflow
  Scenario: Draft → Review → Publish
    # Step 1: Editor creates draft
    Given I am a logged in user with the "editor" user
    When I go to "/admin/content/add"
    And I fill in "Title" with "New Product Launch"
    And I fill in "Body" with "We are launching a new product..."
    And I press "Save as Draft"
    And wait for AJAX to finish
    Then I should see "Status: Draft"

    # Step 2: Editor submits for review
    When I press "Submit for Review"
    And wait for AJAX to finish
    Then I should see "Status: Pending Review"

    # Step 3: Switch to admin for approval
    When I click "Logout"
    And wait for AJAX to finish
    Given I am a logged in user with the "admin" user
    When I go to "/admin/content"
    And wait for AJAX to finish
    Then I should see "Pending Review" in the "New Product Launch" row

    # Step 4: Admin publishes
    When I click "Edit" in the "New Product Launch" row
    And wait for AJAX to finish
    And I press "Publish"
    And wait for AJAX to finish
    Then I should see "Status: Published"

    # Step 5: Verify on public site
    When I go to "/articles/new-product-launch"
    And wait for AJAX to finish
    Then I should see "New Product Launch"
    And I should see "We are launching a new product"

  Scenario: Unpublish content
    Given I am a logged in user with the "admin" user
    When I go to "/admin/content"
    And wait for AJAX to finish
    And I click "Edit" in the "New Product Launch" row
    And wait for AJAX to finish
    And I press "Unpublish"
    And wait for AJAX to finish
    Then I should see "Status: Draft"
```

**How It Works:** The first scenario walks the full lifecycle -- draft, review, publish, public verification -- switching identity twice along the way. Each `Given I am a logged in user with the ... user` mid-scenario re-authenticates as a different role, which mirrors production exactly: different humans, different permissions, one piece of content. The status assertions after every transition (`Status: Draft`, `Status: Pending Review`, `Status: Published`) pin the CMS's state machine, so a workflow module update cannot silently skip or rename a state without going red. The final step is the one teams forget: publishing is not done when the admin sees "Published" -- it is done when the anonymous visitor sees the article, which is why the scenario ends on the public URL. And yes, this looks long for the "one behaviour per scenario" rule. It still honors it: the behaviour under test is the lifecycle itself. Splitting it into five scenarios would test five buttons and lose the handoffs -- which is where the bugs are.

**Pro Tip:** The "Unpublish content" scenario assumes "New Product Launch" already exists, which means it depends on the previous scenario having run. Fine in a linear demo, fragile under `--parallel` where scenario order is not guaranteed. Make it stand alone by seeding the published article first -- through the API steps if your CMS exposes a content API, or by repeating the publish steps -- so every scenario owns its own data.

**At scale:** The publishing workflow is the contract between your team and every editorial team you serve. Tag these scenarios `@workflow @critical` and run them across the whole fleet after every CMS core or security update -- when a distribution update goes out to thirty sites, this is the suite that tells you which two sites have a customized moderation flow that just broke, before their editors find out for you.

---

## Recipe 227: Testing Media Library

**Problem:** Media upload is the most plugin-entangled flow in a CMS -- image styles, file validators, CDN hooks, and editor integrations all pile onto it. So it breaks in interesting ways: a plugin update changes the upload endpoint, and on Friday afternoon marketing is staring at a modal spinner that never stops while the hero image for Monday's campaign sits on their desktop. Upload and insert-into-content are two separate promises; test both.

**Solution:**

```gherkin
Feature: Media Library
  Scenario: Upload an image to the media library
    Given I am a logged in user with the "editor" user
    When I go to "/admin/media"
    And wait for AJAX to finish
    And I press "Upload"
    And wait for the modal to appear
    And I attach the file "test-image.png" to "File Upload"
    And wait for AJAX to finish
    Then I should see "test-image.png"

  Scenario: Insert image into article
    Given I am a logged in user with the "editor" user
    When I go to "/admin/content/add"
    And I fill in "Title" with "Article with Image"
    And I press "Insert Media"
    And wait for the modal to appear
    And I click "test-image.png" in the modal
    And I click "Insert" in the modal
    And wait for the modal to disappear
    And I press "Save as Draft"
    And wait for AJAX to finish
    Then I should see "Content saved"
```

**How It Works:** `I attach the file "test-image.png" to "File Upload"` sets the file directly on the file input -- no operating-system file dialog ever opens, which is what makes this work headless in CI. Keep the fixture image inside the test repo so the path resolves on every machine and every runner. The upload itself is an XHR, so `wait for AJAX to finish` holds the scenario until the request count drains back to zero -- that is the spinner moment, covered. The modal steps wait on the dialog's actual visibility transitions rather than a fixed delay, which matters because media modals load their grid contents asynchronously. The second scenario tests the round trip editors actually care about: open library, pick the image, insert it, save -- proving the editor integration, not just the storage.

**Pro Tip:** Use a tiny fixture image -- a one-kilobyte PNG is plenty. Upload throughput is not the behaviour under test, and a 3MB fixture taxes every run on every project while bloating the repo for nothing.

---

## Recipe 228: Testing CMS User Roles

**Problem:** The client's security review asks "can authors see other people's drafts?" and the honest answer is that nobody knows without clicking around production as four different users. Role permissions are configuration, configuration drifts, and the drift is invisible: a permissions change meant for one role quietly grants Settings access to editors, and it sits there until an audit -- or an incident -- finds it. You want the permission matrix executable, so drift shows up as a red scenario the day it happens.

**Solution:**

```gherkin
Feature: CMS Role Permissions
  Scenario: Admin has full access
    Given I am a logged in user with the "admin" user
    When I go to "/admin"
    And wait for AJAX to finish
    Then I should see "Content"
    And I should see "Users"
    And I should see "Settings"
    And I should see "Structure"

  Scenario: Editor can manage content but not users
    Given I am a logged in user with the "editor" user
    When I go to "/admin"
    And wait for AJAX to finish
    Then I should see "Content"
    And I should not see "Users"
    And I should not see "Settings"

  Scenario: Author can only manage own content
    Given I am a logged in user with the "author" user
    When I go to "/admin/content"
    And wait for AJAX to finish
    Then I should see "My Articles"
    And I should not see "All Articles"

  Scenario: Reviewer can approve but not create
    Given I am a logged in user with the "reviewer" user
    When I go to "/admin/content"
    And wait for AJAX to finish
    Then I should see "Pending Review"
    And I should not see "Add New"
```

**How It Works:** Four scenarios, four roles -- a permission matrix in executable form. Each scenario starts a fresh authenticated session as its role and then asserts in both directions, and the negative assertions are the security half of the test. `I should see "Content"` passes trivially on an over-permissioned site; it is `I should not see "Users"` that catches privilege escalation. Note what this actually tests: the rendered admin UI, meaning what the CMS enforces in front of a real user -- not what the role configuration claims in a settings export. That is the rule "test behaviour, not implementation" doing security work. Belt and braces says also verify that direct URLs like `/admin/users` deny access, because hiding a menu item is not the same as denying the route.

**Pro Tip:** Four logins per run adds up across a fleet. webship-js ships auth-state steps that save and restore the browser's storage state -- log in once per role, save the session, restore it in later scenarios instead of re-driving the login form every time. Keep one scenario per role on the real login path so the login itself stays covered.

**At scale:** This feature is the cheapest security regression you will ever own. Keep one canonical roles feature, keep the role-to-test-user mapping in per-project configuration rather than hardcoded in steps, and run it across every site after any permission-affecting update. A fleet-wide red on "Editor can manage content but not users" the morning after an update is a security finding caught by a feature file -- which is a very good morning, considering the alternative.

---

## Recipe 229: Testing Taxonomy and Categories

**Problem:** Taxonomy is the connective tissue of a CMS -- categories drive menus, listing pages, and feeds, so taxonomy breaks at a distance. Someone renames or deletes a category and three landing pages quietly go empty; free tagging lets "javascript", "Javascript", and "java-script" coexist until the tag cloud is a mess nobody can clean without an audit. Creating terms and attaching them to content are the two operations everything else depends on.

**Solution:**

```gherkin
Feature: Taxonomy Management
  Scenario: Create a new category
    Given I am a logged in user with the "admin" user
    When I go to "/admin/categories"
    And wait for AJAX to finish
    And I press "Add Category"
    And I fill in "Name" with "Technology"
    And I fill in "Description" with "Tech-related articles"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Technology"

  Scenario: Assign tags to an article
    Given I am a logged in user with the "editor" user
    When I go to "/admin/content/add"
    And I fill in "Title" with "Tagged Article"
    And I fill in "Tags" with "javascript, testing, bdd"
    And I press "Save as Draft"
    And wait for AJAX to finish
    Then I should see "Content saved"
```

**How It Works:** The first scenario is straight CRUD on the vocabulary: create the term, save, and assert it appears in the listing -- persistence proven through the same UI an admin would use. The second covers the editor side. `I fill in "Tags" with "javascript, testing, bdd"` types a comma-separated list into the tags field, and on most CMSs that field is an autocomplete widget firing a lookup request per term -- background XHR traffic that the closing `wait for AJAX to finish`, plus the automatic settle webship-js runs after each step, keeps from racing the save. If your taxonomy field creates new terms on the fly, this scenario doubles as a test of that path: save, then check the terms actually landed in the vocabulary.

---

## Real-World Pattern: Editorial Workflow Across Roles

**Problem:** The features above test one role at a time. Real editorial work is a relay: an author drafts, a reviewer approves, a publisher ships -- and the bug you most fear is the one in the *handoff*, where a draft is publicly reachable before anyone approved it. You want a single scenario that walks the content through its whole moderation lifecycle, switching identity at each stage, and proves the gates hold.

**Solution:**

```gherkin
Feature: Editorial Moderation Workflow

  Scenario: A draft moves Draft -> Review -> Published across three roles
    Given I am a logged in user with the "author" user
    When I go to "/node/add/article"
    And I wait for the text "Create Article" to appear
    And I fill in "Title" with "Quarterly Update"
    And I fill in "#edit-body-0-value" with "Draft body copy." by attr
    And I press "Save"
    And I wait for the text "Quarterly Update" to appear
    Then I should see "Draft"

  Scenario: An anonymous visitor cannot reach the unpublished draft
    Given I am an anonymous user
    When I go to "/articles/quarterly-update"
    Then I should see "Access denied"
    And I should not see "Draft body copy."

  Scenario: A reviewer moves it to Published
    Given I am a logged in user with the "reviewer" user
    When I go to "/admin/content/moderated"
    And I wait for the text "Quarterly Update" to appear
    And I click the "Quarterly Update" link
    And I select "Published" from "Change to"
    And I press "Apply"
    And wait for AJAX to finish
    Then I should see "The moderation state has been updated"
```

**How It Works:** Each scenario starts a fresh authenticated session as a different role, and the order tells the story: create as author, prove the gate as anonymous, promote as reviewer. The middle scenario is the one that earns its keep -- `I should see "Access denied"` plus `I should not see "Draft body copy."` proves the unpublished content is sealed both at the route and in the rendered output, the exact privacy promise a moderation workflow exists to keep. The `I am an anonymous user` step clears cookies and starts clean, so it genuinely tests the public view, not a stale editor session. Fill-by-attribute (`#edit-body-0-value ... by attr`) targets the rich-text field directly when its label is not a usable handle.

**At scale:** This three-role relay is the template large distributions lean on -- the Varbase acceptance suite, for instance, structures whole feature groups around "can role X reach state Y?" Keep one moderation feature per content type, map the role names to per-project test users in configuration, and the same scenario protects the editorial gate on every site you run it against. A fleet-wide red on the anonymous-access scenario is a privacy incident caught before release, which is the cheapest place to catch one.

---

## Chapter Summary

- CMS applications have unique content workflow patterns (draft, review, publish, unpublish)
- Test the full publishing lifecycle with multi-user scenarios
- Media library testing covers upload, preview, and insertion into content
- CMS role permissions are more granular than typical apps
- Taxonomy and category management needs CRUD testing
- WYSIWYG/rich text editor testing uses specialized steps
- Content moderation workflows involve multiple user roles
- Negative assertions (I should not see) are the security half of role testing
- One canonical feature set plus per-site LAUNCH_URL scales the same suite across a fleet

Next: Testing Single-Page Applications -- handling client-side routing and state.

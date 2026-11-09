# Chapter 5: Navigation Recipes

---

Navigation is the foundation of every web test. Before you can test a form, a modal, or an API response, you need to get to the right page. This chapter covers every navigation pattern you will encounter.

---

## Recipe 31: Navigating to the Homepage

**Problem:** It is the first day on a new project, and the first thing someone does is hardcode `http://localhost:8080` into a feature file. It works on their laptop. Three weeks later the same suite needs to run against staging in CI, and now there are URLs sprinkled through forty scenarios that all need changing. Every test needs a starting point -- and that starting point needs to live in exactly one place.

**Solution:**

```gherkin
Feature: Homepage Navigation
  Scenario: Visit the homepage
    Given I am on homepage
    Then I should see "Welcome"
```

**How It Works:**

`Given I am on homepage` navigates to the URL configured as `launchUrl` in your `cucumber.js` worldParameters. The `LAUNCH_URL` environment variable overrides it, and the default is `http://localhost:8080`. Your feature files never mention a domain -- the environment decides.

Behind the scenes, the step definition does roughly this:

```javascript
// What webship-js does internally (simplified)
await this.page.goto(this.launchUrl);
await smartSettle(this.page, 3000); // per-navigation settle budget
```

The interesting part is `smartSettle`. webship-js installs three window globals on every page via `addInitScript`: `window.__webshipAjaxCount` (a fetch/XHR wrapper that counts in-flight requests), `window.__webshipPendingTimers` (a `setTimeout` wrapper), and `window.__webshipLastMutation` (a MutationObserver timestamp). After the body is attached, webship-js probes `domcontentloaded` and `networkidle`, then polls those three counters every 50ms. The page counts as settled when AJAX is at zero, no timers are pending, and the DOM has been quiet for at least 250ms. That is why there are no static sleeps anywhere in webship-js -- the framework waits for evidence, not for time.

**At scale:** When one team runs the same suite against twenty sites, `LAUNCH_URL` is the whole trick. The CI job for each project exports its own value -- `LAUNCH_URL=https://staging.project-a.example` -- and the feature files are byte-for-byte identical across projects. The moment a base URL appears inside a `.feature` file instead of an environment variable, that suite stops being shareable. Treat it like a code review rule.

---

## Recipe 32: Navigating to Specific Pages

**Problem:** Your suite needs to visit the about page, the product catalog, the contact form -- a dozen distinct pages per feature. Click-through navigation from the homepage works, but it is slow and it couples every test to the menu structure: redesign the header and thirty scenarios break before they reach the thing they actually test. You want to land directly on the page under test.

**Solution:**

```gherkin
Feature: Page Navigation
  Scenario: Navigate to specific pages
    Given I am on homepage

    When I go to "/about"
    Then I should see "About Us"

    When I go to "/products"
    Then I should see "Our Products"

    When I go to "/contact"
    Then I should see "Contact Us"

    When I go to "/blog"
    Then I should see "Latest Posts"
```

**How It Works:**

`When I go to "/about"` appends the path to your `launchUrl`. If your base URL is `http://localhost:8080`, this navigates to `http://localhost:8080/about`.

The path must start with `/`. webship-js concatenates:

```
launchUrl + path = final URL
http://localhost:8080 + /about = http://localhost:8080/about
```

Each navigation gets the same treatment as the homepage step: `page.goto()` followed by `smartSettle` with its 3000ms per-navigation budget, watching `window.__webshipAjaxCount`, pending timers, and DOM mutations until the page is genuinely quiet. Direct navigation also keeps each scenario testing one behaviour -- if you need five clicks to reach the page, the clicks become five extra ways for the scenario to fail for reasons that have nothing to do with what it asserts. Navigate straight to the page; test the journey separately, once, in its own scenario.

---

## Recipe 33: Navigating to Full URLs

**Problem:** Not everything lives under one domain. The docs are on `docs.example.com`, login goes through an SSO provider on another host, and the marketing site links out to a partner page. The day someone moves the documentation and forgets to update the links, you want a test that notices -- which means your suite occasionally has to leave the comfort of `launchUrl` and visit a full URL.

**Solution:**

```gherkin
Feature: External Navigation
  Scenario: Navigate to a full URL
    Given I am on "https://example.com"
    Then I should see "Example Domain"

  Scenario: Navigate between domains
    Given I am on homepage
    When I go to "/login"
    Then I should see "Sign In"

    # Navigate to a different application
    Given I am on "https://docs.example.com"
    Then I should see "Documentation"
```

**How It Works:**

When you provide a full URL (starting with `http://` or `https://`), webship-js navigates directly to that URL instead of appending to the `launchUrl`. The smart-wait machinery still applies -- the `addInitScript` counters are installed per browser context, not per domain, so AJAX and timer tracking keeps working after the domain hop.

One detail that saves staging environments: webship-js ships with `ignoreHTTPSErrors: true` in `playwright.config.ts`, so a full URL pointing at a staging server with a self-signed certificate will not kill the test the way it would scare a browser user.

Use this for:
- Testing links to external documentation
- Multi-application test suites
- Testing production URLs from staging tests

**At scale:** Hardcoded cross-domain URLs are the second most common reason a shared suite stops being portable (the first is hardcoded `launchUrl`). If several projects share a docs site or an SSO host, put those hosts in environment variables too and keep only genuinely fixed third-party URLs literal in the feature file. And be deliberate about external checks in CI: a partner site having a bad day should not turn your release pipeline red, so tag external-dependency scenarios (for example `@external`) and run them in a separate, non-blocking job.

---

## Recipe 34: Browser Back and Forward

**Problem:** A customer files a bug: "I pressed Back and the page broke." Nobody on the team can reproduce it at first, because nobody tests the back button -- we all navigate forward. In single-page applications especially, client-side routers are notorious for mishandling history: back lands on a blank view, forward double-fires a data fetch, the URL changes but the content does not. If your app has client-side routing, history navigation deserves its own scenario.

**Solution:**

```gherkin
Feature: Browser History
  As a user
  I want to use browser back/forward buttons
  So that I can navigate my browsing history

  Scenario: Navigate back and forward
    Given I am on homepage

    # Build up browsing history
    When I go to "/products"
    Then I should see "Products"

    When I go to "/products/laptop"
    Then I should see "Laptop"

    # Go back
    When I move backward one page
    Then I should see "Products"

    # Go back again
    When I move backward one page
    Then I should see "Welcome"

    # Go forward
    When I move forward one page
    Then I should see "Products"

    # Go forward again
    When I move forward one page
    Then I should see "Laptop"
```

**How It Works:**

- `When I move backward one page` calls `page.goBack()` in Playwright
- `When I move forward one page` calls `page.goForward()`

In a classic multi-page site, each of these triggers a real navigation and gets the full per-navigation `smartSettle` budget. In an SPA, the router intercepts the history event and swaps content client-side -- no page load at all. This is where webship-js's AfterStep hook earns its keep: after every state-changing step it runs `smartSettle(page, 1500)`, so the AJAX the route change kicked off (counted by `window.__webshipAjaxCount`) and the rendering timers behind it are usually settled before your next `Then I should see` runs.

A common bug in SPAs is that the back button does not work correctly -- the URL rewinds but the view does not, or state from the "future" page leaks backward. This recipe catches both, because it asserts on visible content at every hop, not just on the URL.

**Pro Tip:** Browser history navigation can trigger page loads or just URL changes (in SPAs). Add `And wait for AJAX to finish` after navigation if your app loads content dynamically on route changes -- it is redundant when the AfterStep settle already caught everything, but it documents intent and costs nothing when the page is already quiet.

---

## Recipe 35: Page Reload

**Problem:** A user spends ten minutes filling in their profile, hits Save, sees the success message, refreshes the page out of habit -- and everything is back to defaults. The save wrote to memory, not to the server. Bugs like this slip through because manual testers rarely refresh after saving, and automated suites that never reload the page cannot see them either. Reload is the cheapest persistence test you will ever write.

**Solution:**

```gherkin
Feature: Page Reload Behavior
  Scenario: Data persists after reload
    Given I am a logged in user with the "admin" user
    When I go to "/dashboard"
    Then I should see "Dashboard"

    When I reload the page
    Then I should see "Dashboard"
    And I should not see "Sign In"

  Scenario: Form data handling on reload
    Given I am on homepage
    When I go to "/settings"
    And I fill in "Display Name" with "Rajab"
    And I press "Save"
    And wait for AJAX to finish
    Then I should see "Settings saved"

    When I reload the page
    And wait for AJAX to finish
    Then I should see "Rajab" value in the "Display Name" input element
```

**How It Works:**

`When I reload the page` calls `page.reload()` in Playwright. This is equivalent to pressing F5 in the browser. Because a reload is a fresh navigation, webship-js re-runs its `addInitScript` instrumentation -- the AJAX counter, timer tracker, and mutation observer are reinstalled on the new document -- and the full per-navigation `smartSettle` budget applies before the next step runs. Anything that lived only in JavaScript memory is gone after the reload; anything in cookies, localStorage, or the database survives. That boundary is exactly what this recipe tests:

- Session persistence (user stays logged in)
- Saved data appears after refresh
- Client-side state management
- Cache behavior

The second scenario is the one that catches real bugs: fill, save, reload, then assert the field still holds the value. If the success message appeared but the value vanished, you have found an optimistic UI that lied to the user.

---

## Recipe 36: Verifying the Current URL

**Problem:** Login is supposed to land users on the dashboard, and unauthorized visitors are supposed to bounce to the sign-in page. Then one sprint someone changes the auth middleware, the redirect target silently becomes the homepage, and nobody notices until a customer asks why login "forgets where I was going". Redirects are invisible in code review -- the only reliable way to catch a broken one is a test that follows it and checks where it ended up.

**Solution:**

```gherkin
Feature: URL Verification
  Scenario: Login redirects to dashboard
    Given I am on homepage
    When I go to "/login"
    And I fill in "Email" with "admin@example.com"
    And I fill in "Password" with "AdminPass123"
    And I press "Sign In"
    And wait for AJAX to finish
    Then I should see "Dashboard"

  Scenario: Unauthorized access redirects to login
    Given I am on homepage
    When I go to "/admin/settings"
    Then I should see "Sign In"
    And I should not see "Settings"
```

**How It Works:**

The scenarios above verify redirects by checking the **content** of the page you land on. If you see "Dashboard," you know you are on the dashboard page. If you see "Sign In," you know you were redirected to login.

When you also need to pin down the address itself, webship-js 2.0 has dedicated URL assertion steps:

```gherkin
Then I should be on "/dashboard"
Then the url should match "/orders/\d+"
```

`Then I should be on "/path"` compares the current path against what you expect, and `Then the url should match "regex"` lets you assert dynamic URLs -- order IDs, slugs, pagination -- with a pattern instead of an exact string.

So which should you use? Content first, URL second:
- URLs can change during redesigns; behaviour-level content assertions survive them
- Content verification catches more bugs (correct URL but wrong content rendered)
- It follows the BDD principle of testing behavior, not implementation

Reach for the URL steps when the URL **is** the behaviour: canonical redirects for SEO, locale prefixes like `/ar/`, or a payment provider that must return the user to an exact callback path.

**At scale:** Across many projects, the pair "content assertion + `Then I should be on`" is your best defence against the sneakiest redirect failure: the soft redirect to a styled error page. The content check alone can pass ("Welcome" appears on the error page header too); the URL check alone can pass (right address, blank body). Together they are cheap and nearly impossible to fool, which is why redirect scenarios make excellent always-on smoke tests in every project's pipeline.

---

## Recipe 37: Redirect Testing

**Problem:** Your team restructures the blog URLs -- new year, new information architecture. Six months of old links live in newsletters, social posts, and other people's bookmarks. The redirect map gets written once, during the migration, and then nobody touches it. The first time a redirect silently breaks is usually when an annoyed reader emails marketing. A handful of scenarios keeps the promise you made when you changed the URLs.

**Solution:**

```gherkin
Feature: URL Redirects
  As a site administrator
  I want old URLs to redirect to new ones
  So that bookmarks and external links still work

  Scenario: Old blog URL redirects to new structure
    Given I am on homepage
    When I go to "/blog/2024/my-old-post"
    And wait for AJAX to finish
    Then I should see "My Old Post"

  Scenario: HTTP redirects to HTTPS
    Given I am on homepage
    When I go to "/secure-page"
    Then I should see "Secure Content"
    And I should not see "Not Secure"

  Scenario: Trailing slash normalization
    Given I am on homepage
    When I go to "/about/"
    Then I should see "About Us"
```

**How It Works:**

Playwright's `page.goto()` follows HTTP redirect chains automatically -- a 301 from `/blog/2024/my-old-post` to the new URL just happens, the same way it would in a real browser. Your step lands on the final destination, `smartSettle` waits for it to go quiet, and your assertion runs against the page the user would actually see. To pin the destination address as well as the content, add `Then I should be on "/new-path"` after the navigation -- that catches the case where a redirect was changed to point at the wrong (but still rendering) page.

**Pro Tip:** Keep a single `redirects.feature` per project that mirrors your server's redirect map, one scenario per legacy URL family. When someone edits the redirect config, the matching feature change shows up in the same pull request -- and when it does not, the red build asks the question for you.

---

## Recipe 38: 404 and Error Page Testing

**Problem:** Someone follows a dead link and gets a raw server stack trace -- framework version, file paths, the works. Or worse: a blank white page with no way back. Error pages are the part of the site nobody designs twice and nobody tests at all, yet every user eventually sees one. You want proof that your 404 page is helpful, branded, and still has working navigation -- on every deploy, not just the day it was built.

**Solution:**

```gherkin
Feature: Error Pages
  As a user
  I want to see helpful error pages
  So that I know what went wrong and how to fix it

  @regression
  Scenario: 404 page for non-existent URL
    Given I am on homepage
    When I go to "/this-page-does-not-exist-xyz"
    Then I should see "Page Not Found"
    And I should see "The page you are looking for does not exist"
    And I should see "Go to Homepage"

  @regression
  Scenario: 404 page still has navigation
    Given I am on homepage
    When I go to "/nonexistent-page"
    Then I should see "Page Not Found"
    # Navigation should still work
    When I click "Go to Homepage"
    Then I should see "Welcome"
```

**How It Works:**

`page.goto()` does not fail on a 404 response -- the navigation succeeds, the error page renders, and your content assertions take over from there. That is exactly what you want: you are testing the error **page**, not the error **code**. The second scenario is the one teams skip and regret -- it proves the user can escape. A 404 with a dead header menu is a trap, and a trap with your logo on it.

The `@regression` tag marks these scenarios for the full nightly run. Error pages change rarely, so they do not need to run on every pull request -- but they absolutely need to run before a release.

**At scale:** A two-scenario 404 check is one of the highest-value, lowest-cost smoke tests you can standardize across a portfolio. When the same error-page feature file runs on twenty sites, a routing misconfiguration in a deploy -- the kind where suddenly *everything* returns the 404 template, or the 404 returns a 500 -- shows up as twenty failing builds within the hour instead of a support ticket the next morning. Standardize the error-page copy across projects and the feature file needs zero per-project edits.

---

## Recipe 39: Query Parameter Navigation

**Problem:** Search, filters, sorting, pagination -- half of your application's interesting states live in the query string. Clicking through the UI to reach "electronics, sorted by price, page 2" takes five steps and five chances to fail. But the URL *is* the state: navigate straight to it, and the scenario tests what the filtered page shows instead of how many clicks it took to get there.

**Solution:**

```gherkin
Feature: Query Parameter Navigation
  Scenario: Search with query parameter
    Given I am on homepage
    When I go to "/search?q=laptop"
    And wait for AJAX to finish
    Then I should see "Search results for: laptop"

  Scenario: Filtered product listing
    Given I am on homepage
    When I go to "/products?category=electronics&sort=price"
    And wait for AJAX to finish
    Then I should see "Electronics"

  Scenario: Pagination with query parameter
    Given I am on homepage
    When I go to "/blog?page=2"
    And wait for AJAX to finish
    Then I should see "Page 2"
```

**How It Works:**

The query string is just part of the path argument -- webship-js concatenates `launchUrl + "/search?q=laptop"` and navigates. Nothing special happens to the parameters; they reach the server exactly as written. Two things to keep in mind. First, if a parameter value contains spaces or special characters, encode it yourself (`q=gaming%20laptop`) -- the step passes the string through verbatim. Second, deep-linking into a parameterized state is also a real user journey: people bookmark filtered views and share search result links. If `/products?category=electronics&sort=price` renders an unfiltered page, that is a bug your click-through tests would never see.

---

## Recipe 40: Hash/Anchor Navigation

**Problem:** Your FAQ page has a table of contents, your docs have "jump to section" links, and your marketing emails deep-link to `#pricing`. Anchors break quietly -- a heading gets renamed, its `id` changes, and every link pointing at it now scrolls nowhere. The page still loads, so no monitoring catches it. Only a test that follows the anchor and checks what is on screen will.

**Solution:**

```gherkin
Feature: Anchor Navigation
  Scenario: Navigate to page section
    Given I am on homepage
    When I go to "/faq#pricing"
    Then I should see "Pricing FAQ"

  Scenario: In-page anchor links
    Given I am on homepage
    When I go to "/documentation"
    And I click "Installation"
    Then I should see "Installation Guide"

    When I click "Configuration"
    Then I should see "Configuration Options"
```

**How It Works:**

A `#fragment` in the URL never reaches the server -- the browser loads `/faq`, then scrolls to the element whose `id` is `pricing`. There is no second page load and usually no network traffic, which is why webship-js's DOM-quiet check inside `smartSettle` matters here: it watches `window.__webshipLastMutation` rather than network activity, so lazy-rendered sections that mount as you scroll into them still get waited for. Note what the assertion really verifies: that the target content exists and is visible on the page. If you need to prove the page actually *scrolled* there, assert on something only visible in that section's expanded or highlighted state -- or check the fragment with `Then the url should match "#pricing"`.

---

## Recipe 41: Clicking Links

**Problem:** A new hire writes their first scenario, uses `When I click "Submit"` on a button, and watches it fail with "element not found" while the button sits right there on the screen. The click/press distinction trips up everyone exactly once. Meanwhile, the seasoned mistake is different: clicking by CSS class, which works until the next theme update renames everything. You want link clicks that read like English and survive a redesign.

**Solution:**

```gherkin
Feature: Link Interactions
  Scenario: Click text links
    Given I am on homepage
    When I click "About Us"
    Then I should see "Our Story"

  Scenario: Click navigation menu links
    Given I am on homepage
    When I click "Products"
    Then I should see "Our Products"

    When I click "Blog"
    Then I should see "Latest Posts"

    When I click "Contact"
    Then I should see "Get In Touch"

  Scenario: Click a link and verify destination
    Given I am on homepage
    When I click "Read More"
    Then I should see "Full Article"
    And I should not see "Read More"
```

**How It Works:**

`When I click "About Us"` finds an `<a>` element with the text "About Us" and clicks it. The selector strategy:

```
Priority order for finding the element:
1. Link text content: <a>About Us</a>
2. Link title attribute: <a title="About Us">...</a>
3. Link aria-label: <a aria-label="About Us">...</a>
```

**Important distinction:**
- `When I click "Text"` -- for links (`<a>` elements)
- `When I press "Text"` -- for buttons (`<button>`, `<input type="submit">`)

webship-js 2.0 also gives you role-based clicks built on Playwright's `getByRole`:

```gherkin
When I click the "About Us" link
When I click the "Save" button
When I click the "Settings" tab
```

These resolve through the accessibility tree -- the same way a screen reader finds elements -- so they work regardless of whether the "link" is an `<a>`, a `<button>` styled as a link, or a `<div role="link">` someone should not have written. They also double as a quiet accessibility check: if `getByRole` cannot find your link, assistive technology cannot either.

**At scale:** Pick one clicking convention and write it down. On a team maintaining dozens of suites, half the review friction comes from three engineers using three styles for the same action -- `I click "X"`, `I click the "X" link`, and a raw CSS selector. The role-based form is the most resilient across redesigns and across projects with different markup habits, which makes it the best default for a shared convention doc.

---

## Recipe 42: Testing Breadcrumb Navigation

**Problem:** A visitor lands three levels deep from a search engine -- straight onto `/products/electronics/laptops` -- and the breadcrumb trail is their only map of where they are. Then a category gets renamed, the trail still shows the old parent, and clicking it goes to a 404. Breadcrumbs are generated code on most platforms, which means one template bug breaks them on every page at once. Cheap to test, embarrassing to ship broken.

**Solution:**

```gherkin
Feature: Breadcrumb Navigation
  As a user browsing the site
  I want breadcrumbs showing my location
  So that I can navigate back to parent pages

  Scenario: Breadcrumb shows correct hierarchy
    Given I am on homepage
    When I go to "/products/electronics/laptops"
    And wait for AJAX to finish
    Then I should see "Home"
    And I should see "Products"
    And I should see "Electronics"
    And I should see "Laptops"

  Scenario: Clicking breadcrumb navigates to parent
    Given I am on homepage
    When I go to "/products/electronics/laptops"
    And wait for AJAX to finish
    And I click "Electronics"
    And wait for AJAX to finish
    Then I should see "Electronics"
    And I should see "Laptops"
    And I should see "Smartphones"
```

**How It Works:**

The first scenario deep-links into the hierarchy and asserts every level of the trail is present -- if the breadcrumb template drops a level or renders the wrong parent, this fails immediately. The second scenario does what users actually do: click a middle crumb and verify you arrive at the parent listing (seeing both "Laptops" and "Smartphones" proves you are on the Electronics category page, not still on the laptops page). Because breadcrumbs are template-generated, two scenarios on one deep path effectively test the breadcrumb logic for the whole site.

---

## Recipe 43: Testing Pagination Navigation

**Problem:** Page one of the blog always works -- it is the page everyone looks at. The bugs hide at the edges: page 2 shows the same posts as page 1, the "Next" button appears on the last page and leads to an empty list, or a query change quietly breaks the offset math. Nobody clicks to page 10 manually. Your test suite should be the one user who always does.

**Solution:**

```gherkin
Feature: Pagination
  As a user browsing a long list
  I want to navigate between pages
  So that I can see all the items

  Background:
    Given I am on homepage
    When I go to "/blog"
    And wait for AJAX to finish

  Scenario: First page shows initial items
    Then I should see "Post 1"
    And I should see "Next"

  Scenario: Navigate to next page
    When I click "Next"
    And wait for AJAX to finish
    Then I should see "Previous"
    And I should see "Next"

  Scenario: Navigate to specific page number
    When I click "3"
    And wait for AJAX to finish
    Then I should see "Page 3"

  Scenario: Last page has no next button
    When I go to "/blog?page=10"
    And wait for AJAX to finish
    Then I should see "Previous"
    And I should not see "Next"
```

**How It Works:**

The `Background` runs before **each** scenario, so every scenario starts fresh on `/blog` page one -- no scenario depends on where the previous one left off, which keeps each one independently re-runnable. The four scenarios cover the pagination contract: first page renders, Next advances, direct page numbers work, and the last page has no Next. That last one is the assertion teams forget, and it is the one that produces the "empty page 11" support ticket.

Notice the mix of click navigation (`When I click "Next"`) and direct URL navigation (`When I go to "/blog?page=10"`). Clicking tests the control; the URL jump tests the boundary without clicking Next nine times.

**At scale:** The flaky part of pagination tests is never the pagination -- it is the data assumption. "Page 10 is the last page" is true on the project where you wrote the test and false on the other nineteen. Either seed a known content count as part of test setup, or keep boundary assertions structural ("Previous visible, Next absent") rather than tied to a specific page number. Suites that encode content counts from one environment are the ones that go red the first week you reuse them.

---

## Recipe 44: Testing Sidebar/Menu Navigation

**Problem:** The admin panel sidebar is the workhorse navigation of your application -- editors live in it eight hours a day. It is also where permission bugs surface: a menu item that 403s for the role that needs it, a section that vanished after a module update. The Tuesday before release, "can an admin still reach Settings?" should be a green checkmark in the report, not a question someone answers by logging in and looking.

**Solution:**

```gherkin
Feature: Sidebar Navigation
  As a user of the admin panel
  I want to navigate using the sidebar menu
  So that I can access different admin sections

  Background:
    Given I am a logged in user with the "admin" user
    And I go to "/admin"
    And wait for AJAX to finish

  Scenario: Navigate through admin sidebar
    Then I should see "Dashboard"

    When I click "Users"
    And wait for AJAX to finish
    Then I should see "User Management"

    When I click "Content"
    And wait for AJAX to finish
    Then I should see "Content Management"

    When I click "Settings"
    And wait for AJAX to finish
    Then I should see "System Settings"

  Scenario: Active menu item is highlighted
    When I click "Users"
    And wait for AJAX to finish
    Then I should see "User Management"

    When I click "Dashboard"
    And wait for AJAX to finish
    Then I should see "Dashboard"
```

**How It Works:**

The `Background` logs in and lands on `/admin` before each scenario, so the menu walk always starts from a known place. Each click asserts on content unique to the destination section -- "User Management", not just "Users", which also appears in the sidebar itself. That distinction matters: asserting on text that exists in the menu you just clicked is the classic false-positive in navigation tests.

**At scale:** Logging in fresh in every scenario's Background is the single biggest runtime cost in admin-panel suites -- multiply a five-second login by hundreds of scenarios by forty projects and you are paying for hours of CI time that test nothing. webship-js ships auth-state steps that save and restore Playwright's `storageState`: log in once, save the session, and restore it in every scenario after that. Chapter 10 covers the pattern in full; for a sidebar suite like this one it routinely cuts total runtime by half or more.

---

## Navigation Recipes Quick Reference

```
┌───────────────────────────────────────────────────────────────┐
│ What You Want                  │ Step to Use                  │
├───────────────────────────────────────────────────────────────┤
│ Go to homepage                 │ Given I am on homepage       │
│ Go to a page                   │ When I go to "/path"         │
│ Go to full URL                 │ Given I am on "https://..."  │
│ Browser back                   │ When I move backward one page│
│ Browser forward                │ When I move forward one page │
│ Refresh page                   │ When I reload the page       │
│ Verify current URL             │ Then I should be on "/path"  │
│ Verify URL by regex            │ Then the url should match    │
│                                │   "regex"                    │
│ Click a link                   │ When I click "Link Text"     │
│ Click a button                 │ When I press "Button Text"   │
│ Verify content visible         │ Then I should see "text"     │
│ Verify content NOT visible     │ Then I should not see "text" │
│ Verify link URL                │ Then the "X" link should     │
│                                │   contain "/url"             │
└───────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- Navigation is the foundation of every web test
- `Given I am on homepage` → navigates to `launchUrl`
- `When I go to "/path"` → navigates to `launchUrl + /path`
- Use full URLs for external navigation
- Test browser history (back/forward) especially in SPAs
- Test redirects by checking page content first; pin the address with `Then I should be on "/path"` when the URL itself is the behaviour
- Test error pages (404s) to ensure good user experience
- Use `wait for AJAX to finish` after dynamic page transitions
- `click` is for links, `press` is for buttons

Next chapter: Form Testing -- the most recipe-rich chapter in the book.

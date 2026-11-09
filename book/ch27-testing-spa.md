# Chapter 27: Testing Single-Page Applications

---

Single-Page Applications (React, Angular, Vue.js, Svelte) load one HTML page and then dynamically update content using JavaScript. The URL changes but the page never fully reloads. This creates unique testing challenges: client-side routing means `page.goto()` might not trigger a full page load, state management means data persists between views, and lazy loading means components appear asynchronously.

The good news: webship-js handles SPAs well because Playwright understands client-side navigation. Better still -- the smart-wait machinery you met earlier (request counters, timer tracking, DOM-quiet detection) was built with exactly this kind of app in mind. This chapter shows you the patterns that matter.

---

## SPA vs Traditional Page Load

```
┌──────────────────────────────────────────────────────────────────┐
│ Traditional Multi-Page App           Single-Page Application     │
│                                                                  │
│ Click link → Full page load          Click link → JS updates DOM │
│                                                                  │
│ Browser:                             Browser:                    │
│ GET /about → Server renders →        Click "About" →            │
│   Full HTML received →               JS router updates URL →     │
│   CSS, JS, images loaded             JS renders About component  │
│                                      (no server round trip)      │
│                                                                  │
│ Testing impact:                      Testing impact:             │
│ page.goto() triggers load event      page.goto() may not trigger │
│ networkidle is clear signal          standard load events         │
│ DOM is fully fresh                   Previous state may persist  │
│                                                                  │
│ Wait strategy:                       Wait strategy:              │
│ wait for the page to load            wait for AJAX to finish     │
│ (document.readyState)                (content updates via API)   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Recipe 230: Testing SPA Navigation

**Problem:** The product was rewritten as a React SPA and the old navigation tests still pass -- which should worry you, because they only ever assert on the first page. Meanwhile support tickets say "I clicked back and lost my place", and nobody reproduces it manually because everyone tests forward, never backward. In an SPA, navigation is a JavaScript state change wearing a URL costume; the server is not involved, and a whole class of bugs lives in that difference.

**Solution:**

```gherkin
Feature: SPA Navigation
  Scenario: Navigate between SPA routes
    Given I am on homepage
    Then I should see "Welcome"

    When I click "Products"
    And wait for AJAX to finish
    Then I should see "Product Catalog"

    When I click "About"
    And wait for AJAX to finish
    Then I should see "About Us"

    # Browser back should work with client-side routing
    When I move backward one page
    And wait for AJAX to finish
    Then I should see "Product Catalog"

  Scenario: Direct URL navigation in SPA
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    Then I should see "Product Catalog"

    When I go to "/products/laptop-pro"
    And wait for AJAX to finish
    Then I should see "Laptop Pro"
```

**How It Works:** When the test clicks "Products", the router calls `history.pushState` -- the URL changes, the DOM is rewritten in place, and no request for `/products` ever reaches the server. Playwright clicks the real link like a user would, and webship-js's settle machinery takes over: a click is a state-changing step, so the framework automatically runs its smartSettle check afterward, watching the in-page counters it installed before any app code ran -- `window.__webshipAjaxCount` for in-flight fetch/XHR, `window.__webshipPendingTimers` for queued timeouts, `window.__webshipLastMutation` for DOM churn -- and moving on only once requests are at zero, timers are drained, and the DOM has been quiet for 250ms. The explicit `wait for AJAX to finish` adds a firm checkpoint on the data-heavy transitions. `I move backward one page` drives the browser's real back button (`history.back()`), and the assertion after it catches the classic SPA bug where the router updates the URL but not the view. The second scenario is different in kind: `I go to "/products/laptop-pro"` is a full page load, which tests your hosting's deep-link story -- the server must serve the app shell for any route. A 404 there is a hosting-config bug, not an app bug, and this is the only scenario shape that will ever find it.

**Key patterns for SPAs:**
- Always use `wait for AJAX to finish` after navigation (data loads via API)
- Test browser back/forward (common SPA bug: client-side routing breaks history)
- Test direct URL navigation (deep linking must work)
- Test page refresh (SPA must handle server-side rendering or redirect)

**At scale:** Across a portfolio of SPAs -- React on one project, Vue on the next -- this exact feature runs unchanged, because no step ever names the framework. Make it the first feature every new SPA project gets, tagged `@navigation @smoke`. The deep-link and back-button scenarios earn their keep at fleet level: hosting configs differ per project even when the app pattern is identical, and these two scenario shapes are what catch that class of per-project drift.

---

## Recipe 231: Handling Client-Side Routing

**Problem:** A test flakes right after navigation, so the new hire does the intuitive thing on their first PR and adds `wait for the page to load`. It passes on their laptop -- the dev server is slow enough to mask the race -- and fails in CI, where everything resolves instantly. The confusing part: the wait is not waiting. In an SPA the load event fired once, minutes ago, on the first page; every later "page" is a DOM update, and a step that watches load state has nothing left to watch.

**Solution:**

```gherkin
# In a SPA, use these patterns:

# After clicking a navigation link:
When I click "Products"
And wait for AJAX to finish     ← Waits for API data to load
Then I should see "Products"

# After form submission:
When I press "Save"
And wait for AJAX to finish     ← Waits for API response
Then I should see "Saved"

# For direct URL navigation:
When I go to "/dashboard"
And wait for AJAX to finish     ← API calls made on route change
Then I should see "Dashboard"

# NOT this (only works for full page loads):
# And wait for the page to load    ← May resolve immediately in SPA
```

**How It Works:** `wait for the page to load` watches document readiness -- `document.readyState` and the load event. Those signals fire on real navigations, and in an SPA there is exactly one of those: the first. Every later route change resolves the step immediately, which is why it "passes" while the view is still half-rendered. `wait for AJAX to finish` waits on actual work instead: before any application script runs, webship-js injects wrappers around `fetch` and `XMLHttpRequest` (via Playwright's `addInitScript`), so `window.__webshipAjaxCount` tracks every in-flight request, and the step holds until that count reaches zero. A route change that fetches data keeps the count above zero for exactly as long as it matters. And here is the part that surprises people: most of the time you need neither step. Every navigation gets a settle budget of up to 3000ms, every state-changing step is followed by an automatic settle check -- requests at zero, timers at zero, DOM quiet for at least 250ms, polled every 50ms -- and web-first assertions like `Then "sel" should be visible` poll on their own for up to 5 seconds. Keep the explicit AJAX wait for the heavy transitions where you want a named, visible checkpoint in the scenario; a reader should be able to see where the app breathes.

---

## Recipe 232: Testing State Persistence

**Problem:** A router refactor ships, and the cart badge now resets to zero whenever the user visits the About page -- the provider component got unmounted on route change, and nobody noticed because unit tests check the cart logic, not the cart's survival across navigation. State in an SPA lives in JavaScript memory, which means every route change is an opportunity to lose it. The Tuesday before release is a bad time to learn which refactor took that opportunity.

**Solution:**

```gherkin
Feature: SPA State Persistence
  Scenario: Cart persists across navigation
    Given I am on homepage
    When I go to "/products"
    And wait for AJAX to finish
    And I press "Add to Cart" 
    Then I should see "1 item in cart"

    # Navigate to another page
    When I click "About"
    And wait for AJAX to finish
    Then I should see "About Us"
    # Cart count should still show
    And I should see "1 item in cart"

    # Navigate back to products
    When I click "Products"
    And wait for AJAX to finish
    And I should see "1 item in cart"

  Scenario: Form data survives route change (if designed to)
    Given I am a logged in user with the "admin" user
    When I go to "/settings"
    And wait for AJAX to finish
    And I fill in "Display Name" with "Rajab"
    # Navigate away without saving
    When I click "Dashboard"
    And wait for AJAX to finish
    # Navigate back
    When I click "Settings"
    And wait for AJAX to finish
    # Behavior depends on app design: data may or may not persist
```

**How It Works:** Client-side state (Redux, Pinia, context, signals -- the pattern is the same) survives route changes only because the page never reloads, and these scenarios assert that survival directly by re-checking "1 item in cart" after every transition. Each click is followed by webship-js's automatic settle -- requests drained, timers empty, DOM quiet -- so the badge assertion runs after the re-render finishes, not during it. The second scenario is deliberately open-ended, and the comment is the point: whether unsaved form data survives navigation is a product decision, not a testing one. Keep the scenario as a placeholder until the product owner decides, then turn the comment into assertions -- a feature file is exactly the right place for that decision to leave a record everyone can read. One distinction worth keeping sharp: surviving a route change is memory; surviving a reload is storage. If the cart must outlive a refresh, test that separately with `When I reload the page` and assert the persistence layer -- webship-js's cookie steps (`Then a cookie with the name "cart" should exist`) cover the cookie case.

**At scale:** State-persistence bugs are dependency-upgrade bugs -- they arrive when a project bumps its router or state library, and across fifteen SPAs that is happening somewhere every month. Tag these scenarios `@state` and wire them into dependency-update PRs (Renovate, Dependabot), because this is precisely the class of regression that unit tests miss and changelogs never mention.

---

## Recipe 233: Testing Lazy-Loaded Components

**Problem:** The dashboard test passes every time on office wifi and fails one run in five on CI, where a loaded runner takes 1.8 seconds to deliver the chart bundle. Code splitting means parts of your app arrive on demand -- the route renders, then the chunk downloads, then the component mounts, then its data loads -- and a test that assumes "page open means page complete" is betting against the network four times per view.

**Solution:**

```gherkin
Feature: Lazy Loading
  Scenario: Dashboard widgets load progressively
    Given I am a logged in user with the "admin" user
    When I go to "/dashboard"
    And wait for AJAX to finish
    # First widget loads quickly
    Then I should see "Quick Stats"

    # Charts may load later (lazy component + API data)
    And wait for AJAX to finish
    Then I should see "Revenue Chart"
    And I should see "Traffic Chart"

  Scenario: Route-based code splitting
    Given I am on homepage
    # Admin module is lazy-loaded
    When I go to "/admin"
    And wait for AJAX to finish
    # Web-first assertion polls until the lazy chunk renders
    Then "h1" should contain text "Admin Dashboard" within 15 seconds
```

**How It Works:** The first scenario reads as repetitive -- two AJAX waits -- and that repetition is the mechanics of lazy loading: the first wait drains the route's initial data requests, then the chart components mount and fire their own fetches, and the second wait drains those. Two waves of work, two checkpoints. The second scenario is where the old reflex reaches for `wait 1s`, and webship-js 2.0 takes a hard line here: there are no static sleeps anywhere in the framework, and your features should match. A sleep is wrong twice at once -- too long for the fast case (a one-second tax on every green run, forever) and too short for the slow one (the loaded CI runner takes 1.8 seconds and you flake). There is also a subtle reason the settle machinery alone cannot promise this one: lazy chunks arrive via dynamic `import()`, which goes through the browser's module loader, not through `fetch` or XHR -- so the request counter can sit at zero, timers drained, DOM quiet, while the chunk is still on the wire. The web-first assertion bridges exactly that gap: `should contain text` polls the DOM every 100ms and succeeds the instant the heading renders. A fast machine pays a couple hundred milliseconds; a slow one gets the full 15-second budget; nobody pays for time they do not need.

**Pro Tip:** If a chunk regularly needs more than the default 5-second assertion window, that is a performance finding to hand your developers, not a timeout to keep raising. The `within N seconds` clause is for documented exceptions, not a pressure-relief valve.

---

## Recipe 234: Testing API-Driven Components

**Problem:** The user-list test fails with "John Smith not found" and the failure screenshot shows a table full of skeleton loaders -- the component mounted, the assertion ran, and the API response arrived a beat later. Components that fetch on mount put a network round trip between "the page is there" and "the data is there", and every assertion you write is racing it.

**Solution:**

```gherkin
Feature: API-Driven Components
  Scenario: User list loads from API
    Given I am a logged in user with the "admin" user
    When I go to "/admin/users"
    And wait for AJAX to finish
    Then I should see "John Smith"
    And I should see "Jane Doe"

  Scenario: Infinite scroll loads more data
    Given I am on homepage
    When I go to "/feed"
    And wait for AJAX to finish
    Then I should see "Post 1"
    When I scroll to bottom
    And wait for AJAX to finish
    Then I should see "Post 11"
```

**How It Works:** The mount-fetch-render sequence is the heartbeat of an API-driven SPA, and `wait for AJAX to finish` is aimed straight at it: the step holds until `window.__webshipAjaxCount` -- the counter maintained by the fetch/XHR wrappers webship-js injects into every page -- drains to zero, meaning the mount-time requests have all resolved. The infinite-scroll scenario extends the same idea. `I scroll to bottom` brings the sentinel element into view, the intersection observer fires, the next page request goes out, the counter rises -- and the same wait catches it on the way back down. Asserting "Post 11" then proves page two actually rendered, not merely that a request happened. As a second line of defense, web-first assertion steps poll for up to 5 seconds on their own, so a response that lands a beat after the AJAX wait clears still gets caught before a false failure.

**At scale:** The hidden coupling in this recipe is the data: "John Smith" has to exist in every environment you point the suite at, and across many projects that quietly becomes a fixture-management job nobody owns. Two clean ways out: seed what the scenario needs through the API steps in a `Background`, or make the test hermetic by mocking the endpoint with `Given the URL "**/api/users" returns the JSON:` so the UI is tested against data the test controls. Keep a thin `@smoke` layer that hits real data per environment -- and let everything else own its inputs.

---

## Testing Different SPA Frameworks

```
┌──────────────────────────────────────────────────────────────────┐
│ Framework-Specific Considerations                                │
│                                                                  │
│ React:                                                           │
│   • Components re-render on state changes                        │
│   • useEffect hooks trigger API calls after render               │
│   • wait for AJAX catches async useEffect data fetching          │
│                                                                  │
│ Angular:                                                         │
│   • Change detection may delay DOM updates                       │
│   • Observables resolve asynchronously                           │
│   • wait for AJAX handles HttpClient requests                    │
│                                                                  │
│ Vue.js:                                                          │
│   • nextTick needed for DOM updates after data changes          │
│   • Vue Router transitions may need short waits                  │
│   • wait for AJAX handles axios/fetch calls                      │
│                                                                  │
│ Universal Pattern (works for all):                               │
│   When I click "Link"                                            │
│   And wait for AJAX to finish   ← handles async data            │
│   Then I should see "Content"   ← verifies DOM updated          │
│                                                                  │
│ webship-js does not need framework-specific configuration.       │
│ Playwright sees the final DOM regardless of framework.           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- SPAs update the DOM via JavaScript, not full page reloads
- Use `wait for AJAX to finish` instead of `wait for the page to load` for SPAs
- Test browser history (back/forward) -- a common SPA bug
- Test direct URL navigation (deep linking)
- Test state persistence across route changes
- Lazy-loaded chunks arrive outside the request counters -- use web-first assertions, never static sleeps
- webship-js works with React, Angular, Vue, and any SPA framework
- Playwright sees the final DOM regardless of which framework rendered it

Next: Building a Complete Test Suite -- a worked example from start to finish.

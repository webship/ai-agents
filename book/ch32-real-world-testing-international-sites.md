# Chapter 32: Real-World Testing -- International and Enterprise Websites

---

The recipes in this book work for any website. But there is a class of websites that demands extra discipline: international organization sites, government portals, multilingual platforms, and enterprise CMS applications. These sites have unique characteristics -- multiple languages, complex navigation hierarchies, accessibility mandates, content approval workflows, and millions of users who depend on them.

I spent years at Vardot testing exactly this class of site -- Varbase-powered portals for international organizations, governments, and universities, most of them bilingual or trilingual, many of them RTL. The patterns in this chapter are not theory. They are the ones that survived dozens of real projects -- the ones a single QA team can carry from site to site without rewriting everything from scratch.

This chapter takes everything you have learned and applies it to the kind of sites that cannot afford to break. The kind of sites where a broken link, a missing translation, or a failed form costs real credibility -- and where the team that finds the break first should be yours, not a visitor's.

---

## The Anatomy of an International Website

Before you write a single scenario, map the territory. Every international site I have tested -- whatever the CMS, country, or budget -- decomposes into roughly the same anatomy. Treat this map as your test surface: each box is a candidate feature folder, each line inside it a candidate scenario.

```
┌──────────────────────────────────────────────────────────────────┐
│ Typical International Organization Website                       │
│                                                                  │
│  Public-Facing:                                                  │
│  ├── Homepage (multilingual: EN, FR, AR, ES, RU, ZH)           │
│  ├── About Us / Mission                                         │
│  ├── Programs / Projects                                        │
│  ├── Publications / Reports (PDF downloads)                     │
│  ├── News / Press Releases                                      │
│  ├── Events / Calendar                                          │
│  ├── Contact / Office Directory                                 │
│  ├── Search (global site search)                                │
│  ├── Careers / Job Listings                                     │
│  └── Data / Statistics / Interactive Dashboards                 │
│                                                                  │
│  Content Management:                                             │
│  ├── Editorial workflow (Draft → Review → Publish)              │
│  ├── Role-based access (Author, Editor, Publisher, Admin)       │
│  ├── Media library (images, documents, videos)                  │
│  ├── Taxonomy / Tags / Categories                               │
│  └── Translation management                                     │
│                                                                  │
│  Technical Requirements:                                         │
│  ├── WCAG 2.1 AA accessibility compliance                       │
│  ├── RTL language support (Arabic, Hebrew)                      │
│  ├── Responsive design (mobile-first)                           │
│  ├── Performance (CDN, caching, lazy loading)                   │
│  └── Security (CSP headers, cookie consent, data protection)    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Notice that the technical requirements at the bottom cut across everything above them. RTL support is not a page -- it is a property of every page, and the same goes for accessibility and responsive behaviour. That is why this chapter mixes page-level recipes (homepage, contact, publications) with cross-cutting audits (a11y, mobile, multilingual). Product owners write acceptance criteria for the boxes; your suite also has to test the lines between them.

---

## Testing Strategy: The Three-Tier Approach

Not every test is equally important. When you are testing a large site with hundreds of pages, you need to prioritize ruthlessly. Here is the framework:

```
┌──────────────────────────────────────────────────────────────────┐
│ Three-Tier Testing Strategy                                      │
│                                                                  │
│  Tier 1: Critical Path (@critical / @smoke)                      │
│  ─────────────────────────────────────                          │
│  Run on: EVERY push, EVERY deploy                                │
│  Tests:                                                          │
│  ├── Homepage loads in all supported languages                  │
│  ├── Navigation menu works                                      │
│  ├── Search returns results                                     │
│  ├── Contact form submits                                       │
│  ├── Login / authentication works                               │
│  └── Critical content is visible                                │
│  Time: < 5 minutes                                               │
│                                                                  │
│  Tier 2: Functional Integrity (@regression)                      │
│  ─────────────────────────────────────                          │
│  Run on: Before staging, before production deploy               │
│  Tests:                                                          │
│  ├── All page types load correctly                              │
│  ├── Forms validate and submit                                  │
│  ├── Media embeds work (images, videos)                         │
│  ├── Content management CRUD operations                         │
│  ├── Cross-browser verification                                 │
│  └── API endpoints respond correctly                            │
│  Time: < 20 minutes                                              │
│                                                                  │
│  Tier 3: Supplemental (@full / @scheduled)                       │
│  ─────────────────────────────────────                          │
│  Run on: Nightly, weekly, or manually                           │
│  Tests:                                                          │
│  ├── Accessibility audit (all pages)                            │
│  ├── Multilingual content consistency                           │
│  ├── Performance benchmarks                                     │
│  ├── Broken link scanning                                       │
│  ├── Edge cases and error handling                              │
│  └── Visual regression across all breakpoints                   │
│  Time: < 60 minutes                                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Each tier answers a different question for a different audience. Tier 1 answers "can we deploy right now?" -- it is the gate the pipeline blocks on, and it must stay fast enough that nobody is tempted to skip it. Tier 2 answers "did we break anything we already shipped?" -- it runs before a release candidate moves forward, and it is the run a delivery manager wants attached to the release ticket. Tier 3 answers "are we drifting?" -- accessibility erosion, translation gaps, performance creep: the regressions no single deploy causes but every deploy contributes to.

**At scale:** the tags are the contract. When every project in your portfolio uses the same three tags with the same meaning, one question -- "is @critical green?" -- gets a comparable answer on twenty sites. The moment one project quietly redefines @smoke to include a 12-minute regression pack, your cross-project dashboard starts lying. Write the tier definitions down once, centrally, and treat a tag rename like an API break.

---

## Recipe 264: Testing the Homepage

**Problem:** It is 8 a.m., the CMS took its security update overnight, and the first email of the day is from a director's office asking why the homepage shows a maintenance banner. The homepage is the most visited page on any site you run, the first thing stakeholders check, and the last thing anyone consciously tests -- everyone assumes someone else looked at it. You need a homepage check that runs on every deploy, in under a minute, without a human in the loop.

**Solution:**

```gherkin
@critical @smoke
Feature: Homepage
  As a visitor
  I want the homepage to load correctly
  So that I can access the website

  Scenario: Homepage loads with core content
    Given I am on the homepage
    And wait for AJAX to finish
    Then I should see "Welcome"
    And I should see "About"
    And I should see "Contact"

  Scenario: Homepage header and footer are present
    Given I am on the homepage
    And wait for AJAX to finish
    Then I should see a "header" element
    And I should see a "footer" element
    And I should see "Privacy Policy"
    And I should see "Terms of Use"

  Scenario: Homepage navigation works
    Given I am on the homepage
    When I click "About"
    And wait for AJAX to finish
    Then I should see "About Us"
    When I move backward one page
    And wait for AJAX to finish
    Then I should see "Welcome"

  Scenario: Homepage search works
    Given I am on the homepage
    When I fill in "Search" with "annual report"
    And I press "Search"
    And wait for AJAX to finish
    Then I should see "Search Results"
    And I should see "Annual Report"
```

**How It Works:** `Given I am on the homepage` navigates to the `launchUrl` from your world parameters -- set it per environment with the `LAUNCH_URL` env var instead of hardcoding it, so the identical feature file tests local, staging, and production. After navigation, webship-js does not trust `load` or `networkidle` alone: its smartSettle routine polls every 50ms until in-flight AJAX (`window.__webshipAjaxCount`) is zero, pending timers are zero, and the DOM has been quiet for at least 250ms, all within a 3000ms per-navigation budget. The explicit `wait for AJAX to finish` step reads that same counter, which makes it a cheap belt-and-braces line on CMS homepages that lazy-load content blocks after first paint. And the `header`/`footer` assertions are structural on purpose: text can survive a broken layout, but a missing `<footer>` element means the page truly fell apart.

**At scale:** this feature file -- structure unchanged -- is the first thing to copy into every new project you onboard. Keep the scenario names and the shape identical across the portfolio so a failure on project #14 reads exactly like a failure on project #3, and let the pipeline inject `LAUNCH_URL` per environment. Twenty sites, one homepage pattern, zero per-project config edits: that is the difference between maintaining twenty suites and maintaining one pattern twenty times.

---

## Recipe 265: Testing Multilingual Support

**Problem:** Your site speaks six languages, and the language switcher is the most-clicked element in the header. Then a deploy ships a routing change, and the French link starts serving English content under French navigation -- a half-translated page that nobody on the English-speaking dev team notices for two weeks, until a partner organization does, publicly. For most of your audience the language switcher IS the front door, and it deserves the same automated coverage as login.

**Solution:**

```gherkin
@critical @multilingual
Feature: Language Switching
  As an international visitor
  I want to view the site in my language
  So that I can understand the content

  Scenario: Switch to French
    Given I am on the homepage
    When I click "Français"
    And wait for AJAX to finish
    Then I should see "Bienvenue"
    And I should not see "Welcome"

  Scenario: Switch to Arabic (RTL)
    Given I am on the homepage
    When I click "العربية"
    And wait for AJAX to finish
    Then I should see "مرحبا"

  Scenario: Switch to Spanish
    Given I am on the homepage
    When I click "Español"
    And wait for AJAX to finish
    Then I should see "Bienvenido"

  Scenario: Language persists across pages
    Given I am on the homepage
    When I click "Français"
    And wait for AJAX to finish
    When I click "À propos"
    And wait for AJAX to finish
    Then I should see "À propos de nous"
    And I should not see "About Us"

  Scenario: Direct URL for language variant
    Given I am on "/fr"
    And wait for AJAX to finish
    Then I should see "Bienvenue"
    When I go to "/fr/a-propos"
    And wait for AJAX to finish
    Then I should see "À propos"
```

**How It Works:** the dual assertion in the French scenario is the heart of this recipe: `I should see "Bienvenue"` proves the target language rendered, and `I should not see "Welcome"` proves the source language actually left. Half-translated pages pass the first check and fail the second -- this pair is your chimera catcher. The Arabic scenario works because Gherkin strings are plain UTF-8: you write العربية directly in the feature file and Playwright matches it in the DOM like any other text, so keep feature files saved as UTF-8 and RTL needs no special handling at the assertion level. The persistence scenario matters most in practice: language switchers typically set a cookie or URL prefix, and the bug class to fear is the language silently resetting on the second click. The direct-URL scenario (`/fr`, `/fr/a-propos`) covers the visitor who lands from a search engine -- which is most visitors.

**Pro Tip:** the moment your multilingual tests go beyond clicking and reading -- filling forms on translated pages -- remember that `I fill in "Label" with "value"` resolves fields label-first, then by id, name, and placeholder. "Your Name" will not find "Votre nom". Either keep one feature file per language with translated labels, or target fields `by its "data-test" attribute` so one step phrase works in all six languages.

**At scale:** languages multiply everything: twenty projects times four languages is eighty homepage variants. Do not write eighty features. Keep one language-switching feature per project, one scenario per language, all tagged `@multilingual`, and let a nightly `--tags "@multilingual"` run sweep the whole matrix. When a translation regression shows up, the scenario name alone tells the delivery manager which site and which language broke -- no screenshots required.

---

## Recipe 266: Testing Contact Forms on International Sites

**Problem:** The contact form is how a ministry, a journalist, or a funding partner reaches the organization -- and it keeps growing fields nobody re-tests: organization, country, department, subject classification. The failure mode is quiet. The form still renders, visitors still submit, and a misconfigured required field or a broken spam filter swallows every message for a month. Nobody complains, because the people whose messages vanished have no other channel to complain through.

**Solution:**

```gherkin
@regression @forms
Feature: Contact Form
  As a visitor
  I want to submit a contact inquiry
  So that I can reach the organization

  Scenario: Contact page shows all required fields
    Given I am on "/contact"
    And wait for AJAX to finish
    Then I should see "Your Name"
    And I should see "Organization"
    And I should see "Email"
    And I should see "Phone"
    And I should see "Subject"
    And I should see "Message"

  Scenario: Empty form shows validation errors
    Given I am on "/contact"
    When I press "Submit"
    And wait for AJAX to finish
    Then I should see "required"

  Scenario: Submit contact form with valid data
    Given I am on "/contact"
    When I fill in "Your Name" with "Dr. Sarah Johnson"
    And I fill in "Organization" with "Research Institute"
    And I fill in "Email" with "s.johnson@research.org"
    And I fill in "Phone" with "+1 (555) 234-5678"
    And I select "Partnership Inquiry" from "Subject"
    And I fill in "Message" with "We would like to explore collaboration opportunities for the upcoming regional program."
    And I press "Submit"
    And wait for AJAX to finish
    Then I should see "Thank you"

  Scenario: Contact page is accessible
    Given I am on "/contact"
    And wait for the page to load
    Then the page should have no critical accessibility violations
```

**How It Works:** the first scenario asserts every label before anything is filled -- a cheap canary for the "someone edited the form in the CMS" class of regression. The fill steps resolve fields label-first (then id, name, placeholder), exactly the way a human reads a form; if the label-to-input association breaks in the markup, the step fails -- and since that same broken association is an accessibility defect, one failure flags two problems. After `I press "Submit"`, webship-js's AfterStep hook runs smartSettle for up to 1500ms, holding the "Thank you" assertion until the submission round-trip and any confirmation rendering go quiet. The closing scenario runs the axe-core audit on the form page deliberately: forms are where critical accessibility violations concentrate, because every input needs a programmatic label.

### Handling Rate Limiting

Many enterprise sites implement flood control: after one submission, further attempts within a cooldown window are rejected. In CI this bites on the second run -- pipeline A submits the form, pipeline B arrives four minutes later, and the rejection has nothing to do with anyone's code. You cannot assert a single fixed outcome, so write a custom step that accepts either legitimate one:

```javascript
// tests/step-definitions/custom.js
const { Then } = require('@cucumber/cucumber');

Then('I should see a success or rate limit message', async function () {
  const body = await this.page.locator('body');
  const text = await body.textContent();
  const hasSuccess = text.includes('Thank you') || text.includes('Message sent');
  const hasRateLimit = text.includes('too many') || text.includes('try again later');

  if (!hasSuccess && !hasRateLimit) {
    throw new Error('Expected success or rate limit message, got neither');
  }
});
```

```gherkin
  Scenario: Submit form handles rate limiting gracefully
    Given I am on "/contact"
    When I fill in "Your Name" with "Test User"
    And I fill in "Email" with "test@example.org"
    And I fill in "Message" with "Automated testing inquiry"
    And I press "Submit"
    And wait for AJAX to finish
    Then I should see a success or rate limit message
```

The custom step reads the page body once and passes on either acceptable outcome -- success or a flood-control notice -- and fails loudly on anything else, so a genuine breakage (a 500 page, a validation dead-end) still goes red.

**At scale:** flood control is the classic shared-staging trap. When four projects' pipelines hit the same staging server, run B trips the limiter that run A armed, and reviewers start waving away real form failures as "probably the flood control again" -- the most expensive habit a QA team can develop. The dual-outcome step stops the false reds, but the better fix is environmental: ask the platform team to relax flood control on test environments, and keep the dual-outcome step for production smoke checks where you cannot touch server config.

---

## Recipe 267: Testing Navigation Hierarchies

**Problem:** A mega menu three levels deep, breadcrumbs, sidebar navigation, a footer with forty links -- and the information architect just renamed "Programs" to "What We Do" two sprints before launch. Navigation breaks structurally, not visually: the menu still renders beautifully while half its links point at the old URL structure. A visitor who cannot navigate does not file a bug report. They leave.

**Solution:**

```gherkin
@regression @navigation
Feature: Site Navigation
  As a visitor
  I want to navigate through the site structure
  So that I can find the information I need

  Scenario: Main navigation links work
    Given I am on the homepage
    When I click "About"
    And wait for AJAX to finish
    Then I should see "About Us"
    When I move backward one page
    And wait for AJAX to finish
    When I click "Programs"
    And wait for AJAX to finish
    Then I should see "Our Programs"
    When I move backward one page
    And wait for AJAX to finish
    When I click "Publications"
    And wait for AJAX to finish
    Then I should see "Publications"

  Scenario: Footer links work
    Given I am on the homepage
    When I scroll to bottom
    Then I should see "Privacy Policy"
    And I should see "Terms of Use"
    And I should see "Sitemap"
    When I click "Privacy Policy"
    And wait for AJAX to finish
    Then I should see "Privacy"
    And I should see "Data"

  Scenario: Breadcrumb navigation works
    Given I am on "/programs/regional/east-africa"
    And wait for AJAX to finish
    Then I should see "Home"
    And I should see "Programs"
    And I should see "Regional"
    And I should see "East Africa"
    When I click "Programs"
    And wait for AJAX to finish
    Then I should see "Our Programs"
```

**How It Works:** the main-navigation scenario walks click, assert, back, click again -- `I move backward one page` uses browser history instead of re-typing the homepage URL, which is faster and quietly verifies the site survives back-button traffic, something CMS cache layers occasionally mishandle. The breadcrumb scenario starts from a deep URL on purpose: breadcrumbs are generated from the route, and deep links are where route-to-breadcrumb logic breaks first. Between steps, smartSettle keeps assertions from firing while menus are still animating or panels still loading. One detail that saves real debugging time on these sites: when webship-js scrolls an element into view it applies a configurable offset -- 60px by default, tunable via `WEBSHIP_SELECTORS_OFFSET` -- precisely so sticky headers do not sit on top of the element it is about to interact with.

**Pro Tip:** when navigation labels differ per language ("Programs", "Programmes", "البرامج"), define named selectors for the menu containers in `tests/selectors/*.json` and assert the structure with CSS, keeping text assertions inside the language-specific features. Structure is language-independent; labels are not.

---

## Recipe 268: Testing Publications and Document Downloads

**Problem:** For a research organization, publications ARE the product. The annual report launch is coordinated with a press embargo, comms has social posts scheduled for 9:00, and at 8:55 someone discovers the download link returns a 403 because file permissions changed in last night's migration. Listing, metadata, filtering, and the download path all fail independently -- so each one needs its own scenario.

**Solution:**

```gherkin
@regression @publications
Feature: Publications
  As a researcher
  I want to access and download publications
  So that I can use them in my work

  Scenario: Publications page lists reports
    Given I am on "/publications"
    And wait for AJAX to finish
    Then I should see "Annual Report"
    And I should see "Policy Brief"
    And I should see "Technical Guide"

  Scenario: Publication detail page shows metadata
    Given I am on "/publications"
    And wait for AJAX to finish
    When I click "Annual Report 2025"
    And wait for AJAX to finish
    Then I should see "Annual Report 2025"
    And I should see "Published:"
    And I should see "PDF"
    And I should see "Download"

  Scenario: Filter publications by type
    Given I am on "/publications"
    And wait for AJAX to finish
    When I select "Policy Brief" from "Type"
    And wait for AJAX to finish
    Then I should see "Policy Brief"
    And I should not see "Technical Guide"

  Scenario: Search publications
    Given I am on "/publications"
    And wait for AJAX to finish
    When I fill in "Search publications" with "climate"
    And wait for AJAX to finish
    Then I should see "Climate Action Report"
```

**How It Works:** four scenarios, four independent failure modes. The listing scenario proves the index builds. The detail scenario proves the click-through works and the metadata renders -- "Published:", "PDF", "Download" are the three things a researcher scans for before committing to a click. The filter scenario is the subtle one: `I should not see "Technical Guide"` guards against the classic facet bug where the filter updates the URL but not the result list. CMS publication libraries drive these filters with AJAX, which is why each interaction is followed by the AJAX wait -- under the hood webship-js is watching `window.__webshipAjaxCount` drain back to zero before letting the assertion run.

---

## Recipe 269: Testing News and Press Releases

**Problem:** The news section publishes daily -- hourly during a crisis, which is exactly when the communications director is staring at it. Pagination is the silent casualty here: it works on launch day with twelve articles and breaks eight months later when article two hundred pushes a query past a limit nobody knew existed. No human re-tests pagination by hand. Ever.

**Solution:**

```gherkin
@regression @news
Feature: News Section
  As a journalist or visitor
  I want to read the latest news
  So that I stay informed about activities

  Scenario: News page shows recent articles
    Given I am on "/news"
    And wait for AJAX to finish
    Then I should see "Latest News"
    And I should see a ".news-item" element

  Scenario: News article page loads correctly
    Given I am on "/news"
    And wait for AJAX to finish
    When I click "Read more"
    And wait for AJAX to finish
    Then I should see "Published"
    And I should see "Share"

  Scenario: News pagination works
    Given I am on "/news"
    And wait for AJAX to finish
    When I scroll to bottom
    And I click "Next"
    And wait for AJAX to finish
    Then I should see "Latest News"
    And I should see a ".news-item" element
```

**How It Works:** the listing scenario asserts both text ("Latest News") and structure (a `.news-item` element) -- the text survives an empty listing, the CSS assertion does not, and an empty listing is precisely the failure you are hunting. `I click "Read more"` deliberately clicks the first matching teaser rather than a hardcoded headline, because headlines change daily and your feature file must not. The pagination scenario scrolls to the bottom first (paginators on news sections often sit below lazy-loaded content), clicks "Next", and re-runs the same structural assertions -- page two should look like page one, only older.

---

## Recipe 270: Testing Cookie Consent and Privacy

**Problem:** Cookie consent is the one feature where a bug is not a bug -- it is a regulatory finding. If the banner stops appearing, you are setting analytics cookies without consent for every visitor; if "Save" silently ignores the unchecked Marketing box, you are non-compliant while displaying a UI that promises the opposite. Legal finds out from an external audit, and their first question is "when did this break?" -- a question only a test history can answer.

**Solution:**

```gherkin
@critical @privacy
Feature: Cookie Consent
  As a visitor
  I want to manage my cookie preferences
  So that my privacy is respected

  Scenario: Cookie consent banner appears on first visit
    Given I am on the homepage
    Then I should see "cookies"
    And I should see "Accept"

  Scenario: Accept cookies dismisses banner
    Given I am on the homepage
    When I press "Accept"
    And wait for AJAX to finish
    Then I should not see "cookie"

  Scenario: Cookie preferences can be customized
    Given I am on the homepage
    When I press "Manage Preferences"
    And wait for the modal to appear
    Then I should see "Analytics"
    And I should see "Marketing"
    And I should see "Functional"
    When I uncheck "Marketing"
    And I press "Save"
    And wait for AJAX to finish
    Then I should not see "cookie"

  Scenario: Privacy policy is accessible
    Given I am on "/privacy-policy"
    And wait for AJAX to finish
    Then I should see "Privacy Policy"
    And I should see "Data Collected"
    And I should see "Use of the Data"
    And I should see "Cookies"
    And I should see "Changes to the Privacy Policy"
```

**How It Works:** the first-visit scenario depends on starting with a clean cookie jar -- which is what you get, since each scenario runs in a fresh browser context with no stored cookies (to force the state explicitly mid-suite, `Given I am an anonymous user` clears cookies and returns to the launch URL). The preferences scenario is the compliance core: open the modal, uncheck Marketing, save, banner gone. To verify consent beyond the UI, webship-js ships twelve cookie matchers built on Playwright's `context.cookies()` -- after saving you can assert `a cookie with the name "consent" should exist`, or that a marketing cookie is absent after opting out. The banner disappearing proves the UI responded; the cookie assertions prove the choice actually stuck.

**At scale:** privacy regressions are portfolio-wide by nature, because twenty sites usually share one consent module -- when it breaks, it breaks everywhere at once. Keep these scenarios in the @critical tier on every project, not the nightly: consent is the one feature where "we caught it within 24 hours" is still an uncomfortable sentence in front of a regulator. A delivery office that can show a green @privacy run per site per deploy has the audit answer ready before the question arrives.

---

## Recipe 271: Testing the Services and Products Section

**Problem:** Service pages are the pages sales actually sends to prospects -- so a broken one fails in front of exactly the audience you most wanted to impress, attached to a proposal email nobody can unsend. They also change rarely, and "rarely" is how content rot sneaks in: an editor archives a page, a link goes orphan, and the 404 sits there until a prospect finds it before you do.

**Solution:**

```gherkin
@regression @services
Feature: Services Page
  As a potential client
  I want to see available services
  So that I can evaluate offerings

  Scenario: Services page lists all offerings
    Given I am on "/services"
    And wait for AJAX to finish
    Then I should see "Services"
    And I should see "Automated Functional Testing Setup"
    And I should see "Configure Remote CI/CD"
    And I should see "Test Scenario Development"

  Scenario: Service detail page works
    Given I am on "/services"
    And wait for AJAX to finish
    When I click "Contact Us"
    And wait for AJAX to finish
    Then I should see "Contact"
    When I move backward one page
    And wait for AJAX to finish
    Then I should see "Services"
```

**How It Works:** the listing scenario pins the offerings by name -- if someone renames or unpublishes "Configure Remote CI/CD" in the CMS, this fails within the hour instead of next quarter. The detail scenario covers the conversion path: services pages exist to produce "Contact Us" clicks, so that click is the behaviour worth testing, and `I move backward one page` confirms a visitor can return to comparison-shopping without losing the listing. The AfterStep settle gives CMS-rendered blocks time to finish mounting before each assertion, so a slow component does not produce a false red.

---

## Recipe 272: Testing Blog and Content Pages

**Problem:** The blog drives organic traffic, which makes it the section where regressions cost you quietly -- not a crash, just an `<article>` tag that stopped wrapping the content after a theme update, taking the structured data and the search ranking with it. Marketing notices the traffic dip in next month's report and asks engineering what changed three weeks ago. Nobody remembers.

**Solution:**

```gherkin
@regression @blog
Feature: Blog Section
  As a reader
  I want to browse and read blog posts
  So that I can learn from the content

  Scenario: Blog listing page loads
    Given I am on "/blog"
    And wait for AJAX to finish
    Then I should see "Blog"

  Scenario: Blog post renders correctly
    Given I am on "/blog"
    And wait for AJAX to finish
    When I click "Read more"
    And wait for AJAX to finish
    Then I should see a "article" element

  Scenario: Documentation page loads
    Given I am on "/docs"
    And wait for AJAX to finish
    Then I should see "Documentation"
```

**How It Works:** this is deliberately the lightest feature in the chapter -- three scenarios, structural assertions, no content specifics. `I should see a "article" element` is the load-bearing line: it asserts the semantic markup survived the theme update, and semantic markup is what search engines and screen readers actually parse. Clicking "Read more" instead of a dated post title keeps the test stable while posts churn underneath it -- the repo rule applies here: test behaviour, not content. Resist the urge to assert headlines on a blog; that is how a team ends up editing feature files every Monday morning.

---

## Recipe 273: Testing Events and Calendar

**Problem:** Event registration comes with a deadline attached. When the regional conference opens registration and the form is broken, every hour of downtime is a participant emailing the organizers instead -- or quietly giving up. Worse, event sections are seasonal: the registration path can sit unexercised for months between conferences, which is exactly long enough for a CMS upgrade to break it with no one watching.

**Solution:**

```gherkin
@regression @events
Feature: Events
  As a participant
  I want to view and register for events
  So that I can attend

  Scenario: Events page shows upcoming events
    Given I am on "/events"
    And wait for AJAX to finish
    Then I should see "Upcoming Events"

  Scenario: Event detail page shows information
    Given I am on "/events"
    And wait for AJAX to finish
    When I click "View Details"
    And wait for AJAX to finish
    Then I should see "Date"
    And I should see "Location"
    And I should see "Register"

  Scenario: Event registration form works
    Given I am on "/events"
    And wait for AJAX to finish
    When I click "View Details"
    And wait for AJAX to finish
    When I press "Register"
    And wait for AJAX to finish
    Then I should see "Registration"
    When I fill in "Full Name" with "Dr. Maria Santos"
    And I fill in "Email" with "m.santos@university.edu"
    And I fill in "Organization" with "University of Geneva"
    And I press "Submit Registration"
    And wait for AJAX to finish
    Then I should see "Registration Confirmed"
```

**How It Works:** the three scenarios form a funnel: the listing renders, the detail page shows the three facts a participant needs (date, location, how to register), and the registration form takes a submission end to end. The registration scenario chains two click-throughs before any filling, each followed by the AJAX wait so multi-step page loads settle fully -- smartSettle holds each step until pending requests and DOM mutations go quiet. The fill steps resolve "Full Name", "Email", and "Organization" by visible label first, so the test fails when the label-input association fails: a free accessibility check on the form most likely to be rebuilt for every event. Run this funnel on a schedule even between events, against a staging event kept permanently open for exactly this purpose.

---

## Recipe 274: Testing Search Across the Entire Site

**Problem:** On a thousand-page site, search is not a feature -- it is the primary navigation. It is also the feature with the most moving parts: an index that must rebuild as content changes, a query pipeline, facets, language analyzers. When the indexer silently stops, search still "works" -- it just returns last month's site. No error, no crash, and nobody inside the organization notices, because staff navigate by muscle memory, not by search.

**Solution:**

```gherkin
@critical @search
Feature: Global Search
  As a visitor
  I want to search the entire site
  So that I can find specific content quickly

  Scenario: Search returns relevant results
    Given I am on the homepage
    When I fill in "Search" with "annual report 2025"
    And I press "Search"
    And wait for AJAX to finish
    Then I should see "Search Results"
    And I should see "Annual Report"

  Scenario: Empty search shows appropriate message
    Given I am on the homepage
    When I press "Search"
    And wait for AJAX to finish
    Then I should see "enter a search term"

  Scenario: Search with no results shows message
    Given I am on the homepage
    When I fill in "Search" with "xyznonexistentterm123"
    And I press "Search"
    And wait for AJAX to finish
    Then I should see "No results found"

  Scenario: Search filters work
    Given I am on the homepage
    When I fill in "Search" with "policy"
    And I press "Search"
    And wait for AJAX to finish
    And I select "Publications" from "Content Type"
    And wait for AJAX to finish
    Then I should see "Policy"
```

**How It Works:** four scenarios cover the four states every search has -- results, empty query, no results, filtered results. The happy path asserts a specific known document ("Annual Report") rather than settling for "Search Results", and that specificity is what catches the stale-index failure: a generic results page renders fine on a dead index. The empty and no-results scenarios pin the guidance messages, because a blank results page reads as a crash to a visitor. The filter scenario chains a facet selection after the initial query and waits out the AJAX round-trip -- facets are almost always asynchronous, and webship-js releases the step only when its request counter (`window.__webshipAjaxCount`) drains to zero.

**At scale:** search infrastructure is usually shared -- one Solr or Elasticsearch cluster behind many sites -- so a search failure is rarely one project's failure. When the nightly `@search` runs go red on six projects at once, the pattern is the diagnosis: look at the cluster, not the sites. This is one of the strongest arguments for stamping reports with metadata (`--env`, `--app-version`, `--metadata Key=Value` on the report generator), so the platform team can line failures up across projects by time and environment instead of opening twenty HTML files.

---

## Recipe 275: Testing Careers and Job Listings

**Problem:** The careers portal has a peculiar audience: people you want to hire, actively evaluating you. A dead "View Position" link or an empty location filter tells a strong candidate more about your engineering culture than the job ad does. And listings churn constantly -- HR posts and closes positions weekly through the CMS -- so this section breaks far more often than its traffic suggests.

**Solution:**

```gherkin
@regression @careers
Feature: Careers
  As a job seeker
  I want to browse and apply for positions
  So that I can find employment opportunities

  Scenario: Careers page lists open positions
    Given I am on "/careers"
    And wait for AJAX to finish
    Then I should see "Career Opportunities"

  Scenario: Job listing shows details
    Given I am on "/careers"
    And wait for AJAX to finish
    When I click "View Position"
    And wait for AJAX to finish
    Then I should see "Job Description"
    And I should see "Requirements"
    And I should see "Apply"

  Scenario: Filter jobs by location
    Given I am on "/careers"
    And wait for AJAX to finish
    When I select "Geneva" from "Location"
    And wait for AJAX to finish
    Then I should see "Geneva"
```

**How It Works:** the scenarios assert the stable skeleton and ignore the churn: the page frames its purpose ("Career Opportunities"), a click-through reveals the three blocks every posting needs (description, requirements, a way to apply), and the location filter narrows. Notice what is NOT asserted -- no job titles, no listing counts -- because content that changes weekly would turn this into a feature file someone has to edit weekly. The filter scenario's closing `I should see "Geneva"` is intentionally loose for the same reason; tighten it only if your filter develops a history of lying.

---

## Recipe 276: Testing Data and Statistics Dashboards

**Problem:** The statistics dashboard is the page your organization cites in press releases -- and the hardest page you own to test. It renders client-side from API calls, so the page can "load" perfectly while every chart sits empty because one endpoint timed out. Layout checks pass, visitors see no data, and the data team hears about it when a journalist asks why the dashboard is blank.

**Solution:**

```gherkin
@regression @data
Feature: Data Dashboard
  As a researcher
  I want to access interactive data
  So that I can analyze trends

  Scenario: Dashboard page loads with visualizations
    Given I am on "/data"
    And wait for AJAX to finish
    Then I should see "Dashboard"
    And I should see a ".chart-container" element

  Scenario: Dashboard filters update data
    Given I am on "/data"
    And wait for AJAX to finish
    When I select "2025" from "Year"
    And wait for AJAX to finish
    Then I should see "2025"

  Scenario: Data export works
    Given I am on "/data"
    And wait for AJAX to finish
    Then I should see "Download"
    And I should see "CSV"
```

**How It Works:** asserting `I should see a ".chart-container" element` is the right altitude for canvas- and SVG-rendered charts: you cannot cheaply assert what a chart draws, but you can assert that its container mounted -- which fails the moment the rendering JavaScript throws. This is also where webship-js's settle logic earns its keep: dashboards fire several API calls and animate on arrival, and smartSettle holds each step until in-flight requests and DOM mutations go quiet instead of asserting against a half-drawn page. Pair this feature with JS error capture -- `WEBSHIP_JS_ERROR_MODE=fail` suite-wide, or the `@js-fail` tag on just this feature -- because on a dashboard a console exception usually IS the bug, even when the page looks superficially fine.

**Pro Tip:** for deterministic dashboard tests, take the data warehouse out of the equation: `Given the URL "**/api/stats" returns the JSON:` intercepts matching requests through Playwright's `page.route()` glob matching and serves your fixture instead. The filter scenario then tests the dashboard's behaviour, not the warehouse's mood. Keep one unmocked scenario as a live-data canary.

---

## Recipe 277: API Testing for International Sites

**Problem:** The same content feeds the website, the mobile app, and three partner organizations syndicating your publications -- all through one API. Browser tests will never catch the field that quietly disappeared from the JSON payload, because the website's frontend was updated to match. The mobile app was not. Its users get a crash, and the app-store reviews arrive before the bug report does.

**Solution:**

```gherkin
@regression @api
Feature: Public API
  Background:
    Given the API base URL is "https://api.example.org"

  Scenario: API returns list of publications
    When I send a GET request to "/api/publications"
    Then the API response code should be 200
    And the response should be valid JSON
    And the JSON response should have property "data"

  Scenario: API returns single publication
    When I send a GET request to "/api/publications/1"
    Then the API response code should be 200
    And the JSON response should have property "title"
    And the JSON response should have property "published_date"
    And the JSON response should have property "language"

  Scenario: API supports language filter
    When I send a GET request to "/api/publications?lang=fr"
    Then the API response code should be 200
    And the response should be valid JSON

  Scenario: API returns events
    When I send a GET request to "/api/events"
    Then the API response code should be 200
    And the response should be valid JSON

  Scenario: API returns countries data
    When I send a GET request to "/api/countries"
    Then the API response code should be 200
    And the response should be valid JSON
    And the JSON response should have property "data"

  Scenario: API 404 for unknown resource
    When I send a GET request to "/api/nonexistent"
    Then the API response code should be 404

  Scenario: API with authentication
    Given I am authenticating as "api_user" with "api_key_123" password
    When I send a GET request to "/api/admin/stats"
    Then the API response code should be 200
    And the JSON response should have property "total_visitors"
```

**How It Works:** `Given the API base URL is "..."` in the Background sets the base for every request in the feature, and the requests go through axios rather than the browser -- so these scenarios run in milliseconds with no page involved. The property assertions are contract checks: `the JSON response should have property "title"` fails the moment a serializer change drops a field a consumer depends on, whether or not the website still renders. The language-filter scenario (`?lang=fr`) is the international-site special -- translated content endpoints break independently of the default language, and only an explicit per-language request catches it. The 404 scenario pins error behaviour, because an API that answers unknown routes with a 200 HTML error page (a classic CMS misconfiguration) breaks every consumer that parses JSON. The authenticated scenario proves the wall stands in one direction; pair it with an unauthenticated request to the same admin endpoint asserting a 401 or 403 to prove it stands in the other.

**At scale:** API features are the cheapest tests in your portfolio -- no browser, no screenshots, milliseconds per scenario -- so they are the tier you can run on every push across twenty projects without anyone asking about CI minutes. Keep the Background's base URL as the single per-environment line the pipeline owns, and tag the suite `@api` so it can run standalone (`npx cucumber-js --tags "@api"`) as a fast first gate before any browser container spins up.

---

## Recipe 278: Accessibility Audit for International Sites

**Problem:** For public institutions, WCAG 2.1 AA is procurement criteria, legislation, or both -- and the accessibility statement on your site is a claim auditors test. The cruel part is how accessibility regresses: one unlabeled input this sprint, one low-contrast button the next, each individually invisible, until the annual external audit drops a forty-page findings document on the delivery manager's desk and every line item becomes rush-priority at once.

**Solution:**

```gherkin
@a11y @scheduled
Feature: Accessibility Compliance
  Scenario: Homepage is accessible
    Given I am on the homepage
    And wait for the page to load
    Then the page should have no critical accessibility violations

  Scenario: Contact page is accessible
    Given I am on "/contact"
    And wait for the page to load
    Then the page should have no critical accessibility violations

  Scenario: Publications page is accessible
    Given I am on "/publications"
    And wait for AJAX to finish
    And wait for the page to load
    Then the page should have no critical accessibility violations

  Scenario: News page is accessible
    Given I am on "/news"
    And wait for AJAX to finish
    And wait for the page to load
    Then the page should have no critical accessibility violations

  Scenario: Keyboard navigation works on homepage
    Given I am on the homepage
    When I press the "Tab" key
    Then the focused element should be visible
    When I press the "Tab" key
    Then the focused element should be visible
    When I press the "Enter" key
    And wait for AJAX to finish
    Then I should not see "error"
```

**How It Works:** `the page should have no critical accessibility violations` injects axe-core into the live page and runs its rule engine against the rendered DOM -- real markup, real contrast, real ARIA relationships, not a static lint of templates. Critical violations (missing form labels, broken ARIA references, keyboard traps) are the ones that make a page unusable with assistive technology, which is why this threshold makes a sane CI gate where a full audit would drown you in noise; when you are ready for the stricter bar, the same engine backs `the page should pass an accessibility audit at level "AA"`. The keyboard scenario covers what no rule engine can: whether tabbing actually moves focus somewhere visible. And the `wait for the page to load` lines before each audit are not decoration -- auditing a page whose components are still mounting produces phantom violations, and settling first prevents exactly that.

**Pro Tip:** when an audit fails in CI, reproduce locally and add `Then print accessibility violations` after the failing step -- you get the violation list with selectors in your console instead of reverse-engineering it from a report.

**At scale:** run the `@a11y` suite nightly on every project and aggregate the results in one place -- this is the quality signal management actually understands, because "zero critical violations across all twenty sites" maps directly onto the compliance language they report upward. Stamp each run with the report generator's metadata flags (`--env`, `--app-version`, `--metadata Project=name`) so nightly reports are comparable across the portfolio, and treat a new critical violation like a failed build rather than a backlog item. The whole point is to never again meet accessibility debt forty pages at a time.

---

## Recipe 279: Testing Mobile Responsiveness

**Problem:** In many of the regions international organizations serve, mobile is not a viewport -- it is the internet. A contact form that works on desktop but hides its Submit button behind a sticky footer at 375px wide is, for a majority of your real audience, simply broken. Yet every developer and every stakeholder reviews the site on a large monitor, so mobile regressions sail through every human check you have.

**Solution:**

```gherkin
@mobile @regression
Feature: Mobile Experience
  Scenario: Homepage loads on mobile
    Given I set the viewport to mobile
    And I am on the homepage
    And wait for AJAX to finish
    Then I should see "Welcome"

  Scenario: Mobile navigation menu works
    Given I set the viewport to mobile
    And I am on the homepage
    And wait for AJAX to finish
    When I click "Menu"
    And wait for AJAX to finish
    Then I should see "About"
    And I should see "Programs"
    And I should see "Contact"
    When I click "Contact"
    And wait for AJAX to finish
    Then I should see "Contact"

  Scenario: Contact form works on mobile
    Given I set the viewport to mobile
    And I am on "/contact"
    And wait for AJAX to finish
    When I fill in "Your Name" with "Mobile User"
    And I fill in "Email" with "mobile@example.com"
    And I fill in "Message" with "Testing from mobile viewport"
    And I press "Submit"
    And wait for AJAX to finish
    Then I should see "Thank you"

  Scenario: Tables are readable on mobile
    Given I set the viewport to mobile
    And I am on "/data"
    And wait for AJAX to finish
    Then I should see "Dashboard"
```

**How It Works:** `I set the viewport to mobile` resizes the page to a phone-sized viewport, and everything after it runs the same steps as desktop -- that symmetry is the point. webship-js carries a named breakpoint registry for exactly this work (xs at 375x667 up through xxxl at 1920x1080, with xl at 1200x900 as the default), and you can redefine the whole set as JSON via `WEBSHIP_SELECTORS_BREAKPOINTS`. The menu scenario exercises the hamburger pattern -- navigation collapsed behind a "Menu" toggle -- and the form scenario re-runs the full fill-and-submit flow because mobile is where elements actually overlap: Playwright refuses to click an element that another element covers, so a sticky banner sitting on the Submit button fails here while passing on desktop. That refusal is not flakiness. It is the bug you wanted caught.

**At scale:** breakpoints are a portfolio convention, not a per-project setting. Define them once -- the same `WEBSHIP_SELECTORS_BREAKPOINTS` JSON exported by your shared CI configuration -- so "mobile" means 375x667 on every project and a designer's "which width failed?" has one answer everywhere. And do not double your whole suite for mobile: run everything at desktop, plus a curated `@mobile` subset (navigation, forms, anything sticky-positioned) per project. That covers the genuinely viewport-sensitive behaviour for roughly 15% of the runtime -- a number your DevOps team will accept without a meeting.

---

## Complete Project Structure for an International Site

Here is how the whole chapter lands on disk. Two conventions carry this structure: feature files are named `area--behaviour.feature`, so a failing filename alone tells you what broke before you open anything, and folders mirror the site's information architecture, so a product owner can find the scenarios for "their" section without a guided tour.

```
organization-tests/
├── package.json
├── cucumber.js
├── playwright.config.ts
│
├── tests/
│   ├── features/
│   │   ├── homepage/
│   │   │   ├── homepage--page-load.feature
│   │   │   ├── homepage--navigation.feature
│   │   │   └── homepage--search.feature
│   │   ├── contact/
│   │   │   ├── contact--page-load.feature
│   │   │   ├── contact--form-validation.feature
│   │   │   └── contact--form-submission.feature
│   │   ├── publications/
│   │   │   ├── publications--listing.feature
│   │   │   └── publications--filters.feature
│   │   ├── news/
│   │   │   └── news--listing.feature
│   │   ├── multilingual/
│   │   │   └── language-switching.feature
│   │   ├── privacy/
│   │   │   ├── cookie-consent.feature
│   │   │   └── privacy-policy.feature
│   │   ├── api/
│   │   │   ├── publications-api.feature
│   │   │   └── events-api.feature
│   │   └── accessibility/
│   │       └── a11y-audit.feature
│   │
│   ├── step-definitions/
│   │   └── custom.js
│   │
│   └── support/
│       └── hooks.js
│
├── tests/reports/
└── .github/workflows/test.yml
```

Keep `custom.js` honest: it should hold only steps genuinely specific to this site, like the rate-limit step from Recipe 266. The moment a custom step proves useful on a second project, move it out of the project and into a shared package -- copy-pasted step definitions are how twenty suites drift into twenty dialects.

---

## Running the Suite with Tags

Tags are the API of your test suite -- they are how pipelines, schedules, and humans select what runs. These commands assume the taxonomy from the three-tier strategy:

```bash
# Critical path -- run on every push
npx cucumber-js --tags "@critical"

# Full regression -- run before deploy
npx cucumber-js --tags "not @scheduled and not @wip"

# Accessibility audit -- run nightly
npx cucumber-js --tags "@a11y"

# Mobile tests only
npx cucumber-js --tags "@mobile"

# Multilingual tests
npx cucumber-js --tags "@multilingual"

# API tests (no browser needed)
npx cucumber-js --tags "@api"

# All tests, parallel execution
npx cucumber-js --parallel 4 --tags "not @wip"
```

**At scale:** standardize this exact command set across every project. When all pipelines in the portfolio call the same handful of commands with the same tags, the platform team maintains one CI template instead of twenty bespoke pipelines -- and webship-js ships starter configs for thirteen CI providers (GitHub Actions, GitLab CI, Jenkins, Azure Pipelines, and friends), so the template is mostly written for you. The `--parallel 4` flag is your nightly cost lever: workers multiply CPU but divide wall-clock time, and on a farm running forty suites a night, wall-clock is what you pay for.

---

## The Testing Advantage for International Organizations

If you ever need to justify the testing budget to a steering committee, this is the slide:

```
┌──────────────────────────────────────────────────────────────────┐
│ Why Automated Testing Matters for International Sites            │
│                                                                  │
│  Without automated tests:                                        │
│  ├── A broken translation goes unnoticed for weeks              │
│  ├── A form stops working after a CMS update                    │
│  ├── Cookie consent breaks and violates GDPR                    │
│  ├── Navigation links point to 404 pages                        │
│  ├── Mobile users cannot access critical services               │
│  └── Accessibility regressions exclude disabled users           │
│                                                                  │
│  With automated tests:                                           │
│  ├── Every page is verified on every deploy                     │
│  ├── Forms are tested with valid and invalid data               │
│  ├── Privacy compliance is continuously verified                │
│  ├── Navigation is tested across all languages                  │
│  ├── Mobile experience is verified automatically                │
│  └── Accessibility is audited regularly                         │
│                                                                  │
│  The cost of a bug on a site serving millions of people          │
│  is not measured in dollars. It is measured in trust.            │
│                                                                  │
│  Automated testing is not a luxury for international sites.     │
│  It is the foundation of reliability.                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- International websites have unique requirements: multilingual, accessible, secure, high-traffic
- Use a three-tier strategy: Critical (every push), Regression (before deploy), Supplemental (nightly)
- Test the homepage first -- it is the front door
- Test language switching and content persistence across navigation
- Handle rate limiting with custom steps that accept multiple outcomes
- Test cookie consent for privacy compliance
- Verify accessibility with axe-core integration
- Test mobile responsiveness -- mobile is the primary device in many regions
- API tests complement browser tests for data endpoints
- Use tags to organize and run test subsets efficiently
- The cost of a bug on an international site is measured in trust, not just money

---

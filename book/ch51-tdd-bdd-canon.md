# Chapter 51: Foundations -- Wisdom From the TDD and BDD Canon

---

Every test you write rests on the shoulders of people who wrote tests before you. The vocabulary of "red, green, refactor", of "given-when-then", of "the test pyramid", did not emerge from a vacuum. It came from a generation of practitioners who wrote books, ran experiments in real codebases, and argued in long email threads about what testing actually means. This chapter pulls the most useful ideas from that canon and pins them, recipe by recipe, to webship-js practice.

If you've read Kent Beck's *Test-Driven Development by Example*, Dan North's BDD essays, Gojko Adzic's *Specification by Example*, Wynne and Hellesøy's *The Cucumber Book*, Freeman and Pryce's *Growing Object-Oriented Software, Guided by Tests*, or Lisa Crispin and Janet Gregory's *Agile Testing* -- you'll recognise these ideas immediately. If you haven't, this chapter is the shortest route to the gist of each. Either way, the goal is the same: ground your day-to-day webship-js work in the rest of the field.

Appendix D lists the full bibliography. This chapter is the operational summary.

---

## The Five Strands of the Canon

<!-- diagram: bdd-flow -->

Five lineages converge in modern acceptance testing:

```
TDD (Kent Beck, 1999)
   ↓
BDD (Dan North, 2006) ──→ webship-js
   ↑
ATDD / SbE (Gojko Adzic, 2009-2011)
   ↑
Cucumber lineage (Hellesøy, Wynne, 2008+)
   ↑
GOOS (Freeman & Pryce, 2009)
```

TDD is the foundational rhythm. BDD reframed TDD in user-language. ATDD/SbE pulled stakeholders into the same conversation. The Cucumber lineage gave the rhythm a syntax. GOOS extended TDD to whole-system design. Webship-js sits on top of all five.

Read the canon and the patterns you've used without thinking suddenly carry names: *triangulation*, *walking skeleton*, *test doubles*, *outside-in*, *living documentation*, *the saff squeeze*.

---

## Recipe 701: Red, Green, Refactor (Beck, 2002)

### Idea
Kent Beck's TDD rhythm:

1. **Red** -- Write a failing test that captures the next bit of behaviour you want.
2. **Green** -- Write the smallest amount of production code that makes the test pass.
3. **Refactor** -- Improve the code without changing what it does. The tests guarantee you haven't broken anything.

The rhythm forces you to keep tests and code in lockstep. The tests are not an afterthought; they drive the design.

### Webship-js translation
Here is the moment this rhythm earns its keep. A developer ships the coupon feature on Thursday. QA writes the scenario on Friday. It passes on the first run, everyone smiles, and nobody notices it asserts the wrong discount -- the scenario never had the chance to fail, so it never proved it could catch anything. Red first is not a ritual. It is the only evidence you ever get that the test can detect the bug it was written for.

For acceptance tests, the rhythm becomes:

1. **Red** -- Write a `Scenario:` that fails (the feature doesn't exist or doesn't work yet).
2. **Green** -- Implement the feature until the scenario passes.
3. **Refactor** -- Clean up the implementation, knowing the scenario will catch regressions.

```gherkin
# Step 1: Red -- this fails because /coupon doesn't exist yet
Scenario: Apply a coupon
  Given I am on "/cart"
  When I fill in "code" with "SAVE10"
  And I click "Apply"
  Then eventually I should see "Discount: -$10.00" within 5 seconds
```

The scenario is the executable specification. The development task is "make it green."

One detail makes red trustworthy in webship-js: there are no static sleeps anywhere in its 411 built-in steps. `eventually I should see ... within 5 seconds` polls the page's text every 100ms until the deadline, and after every state-changing step an AfterStep hook runs `smartSettle(page, 1500)` -- waiting until pending fetch/XHR calls hit zero, pending timers hit zero, and the DOM has been quiet for at least 250ms. So when this scenario goes red, it is because the behaviour is missing, not because the assertion fired while an AJAX call was still in flight. An honest red is what makes green mean something.

---

## Recipe 702: Triangulation (Beck)

### Idea
When the simplest implementation that passes one test is "return a hard-coded value", you don't yet have enough tests. Write a second test that forces the implementation to generalise. The technique is **triangulation** -- two examples pin down the behaviour where one was ambiguous.

### Webship-js translation
Every team has shipped the hard-coded version of something at least once -- the discount that was literally `total - 10` because the only example used a $100 cart. The test stayed green for a year. The first $50 cart in production produced a negative total, and the postmortem asked how the tests missed it. They didn't miss it. There was only one example, and one example cannot pin down a rule.

```gherkin
Scenario: 10% coupon discounts a $100 cart by $10
  ...

Scenario: 10% coupon discounts a $200 cart by $20
  ...
```

The first scenario alone could be satisfied by "always subtract $10." The second forces the implementation to multiply by the cart total. Two examples, one inferred rule.

In Gherkin, the cheap way to triangulate is a `Scenario Outline` with an `Examples` table -- one structure, several rows, each row a fresh chance to expose a hard-coded answer. Choose the rows deliberately: $100 and $100.00 exercise the same code path and triangulate nothing. $100 and $237.50 force the real multiplication into existence.

### Recipe
When a test is too easy to pass with a hard-coded answer, add a second scenario with a different input -- different enough that a faked implementation would fail it. The pair triangulates the behaviour.

---

## Recipe 703: BDD as Translation (North, 2006)

### Idea
Dan North coined BDD because he kept finding developers stuck on the word "test." If you write "test something", you fixate on the implementation; if you write "describe a behaviour", you stay at the user-language level. BDD's contribution was not a tool but a vocabulary: *given*, *when*, *then*; *features*, *scenarios*, *examples*.

### Webship-js translation
You can spot a suite that lost this translation in one scroll: scenarios full of POSTs, payloads, and CSS classes that the product owner stopped reading months ago. Once that happens, sign-off becomes an act of trust -- which means it is not really sign-off. Every feature file should be the vocabulary in action:

```gherkin
Feature: Coupon discounts
  As a shopper
  I want to enter a coupon code
  So that I get the advertised discount

  Scenario: Apply a coupon
    Given I am on "/cart"
    When I fill in "code" with "SAVE10"
    Then eventually I should see "Discount applied" within 2 seconds
```

The header section is North's "user story" -- role, action, value. The scenario is the example that proves the story.

Webship-js bakes the same translation into its step engine. `When I fill in "code" with "SAVE10"` resolves "code" label-first -- the step looks for the visible label before falling back to id, name, or placeholder. You write the word the shopper sees on the screen; the step finds the input behind it. That is North's idea running at selector depth: the scenario speaks user, the framework speaks DOM, and the translation happens in the one place equipped to do it.

### Recipe
Keep the user-language tier in feature files. Never let implementation vocabulary leak in. If you write `When I post to /api/coupons/apply`, you've lost the user lens. Rewrite as `When I click "Apply"`.

---

## Recipe 704: The Ubiquitous Language (Evans, 2003; North, 2006)

### Idea
Eric Evans introduced "ubiquitous language" in *Domain-Driven Design*: the team should use the same words in conversation, documentation, and code. Dan North made BDD a vehicle for that language -- every Gherkin step uses domain terms, not implementation jargon.

### Webship-js translation
The drift is slow and nobody decides it. Marketing renames "customer" to "member" in March. The UI follows in April. By June your steps still say customer, the product owner's criteria say member, and every refinement session starts with two minutes of translation. Same system, two dialects. Your custom steps and feature files should use the vocabulary stakeholders actually use:

```gherkin
# Good (domain language)
When the customer applies the coupon "SAVE10"
Then the order summary should show a "$10.00" discount

# Bad (implementation language)
When the user POSTs to /api/orders/42/coupons
Then the response body should have property "discount":1000
```

The language layer goes deeper than feature files. Webship-js keeps a named-selector registry -- JSON files in `tests/selectors/` mapping a human name to a CSS or XPath selector, auto-loaded through `worldParameters.selectors.files`. Name those entries in domain terms too ("order summary", "discount banner"), and the ubiquitous language extends down into the one place implementation detail is allowed to live.

**At scale:** across twenty projects, vocabulary drift multiplies into a review tax -- a QA lead moving between suites has to relearn what each team calls the same concept. Keep one shared glossary for cross-project terms and let each project add domain words, not synonyms. When a shared step library exists, the glossary is effectively its API documentation.

### Recipe
Audit your step library quarterly for terms that drifted away from the domain. "User", "customer", "shopper" should match what the product team says. If marketing rebrands "customer" to "member", update the steps -- and the selector names with them.

---

## Recipe 705: Three Amigos (Adzic, 2011)

### Idea
Gojko Adzic's *Specification by Example* describes the **Three Amigos** meeting: a product person, a developer, and a tester (or QA, or designer) sit together and turn a user story into concrete examples. The examples become the acceptance scenarios. Conversation happens once; the scenarios encode the agreement.

### Webship-js translation
The expensive version of skipping this meeting is the sprint demo. The developer shows the coupon flow, the product owner says "that's not what I meant -- it should apply before shipping is calculated, not after", and three days of work plus the scenarios that faithfully tested the wrong behaviour go back into the sprint. Thirty minutes of conversation would have caught it on a whiteboard.

For every meaningful feature, hold a Three Amigos session. Outputs:

- 3-7 concrete scenarios in Gherkin
- A shared understanding of edge cases (zero-quantity coupon, expired coupon, currency conversion)
- A list of open questions for the product owner

The scenarios drive both development (developer's red-green-refactor) and verification (tester's coverage check). One artefact, three uses.

**At scale:** when one QA team supports many product teams, Three Amigos is what keeps acceptance criteria comparable across projects -- same format, same concreteness, same idea of what counts as an example. Rotate which QA engineer attends which project's sessions, and the conventions travel with them. For a PMO, the session's output is the first reviewable evidence that a story was actually understood before anyone built it.

### Recipe
Run Three Amigos for any feature larger than a one-line bug fix. Schedule 30 minutes. Bring a laptop and write the scenarios live. Save them in the repo as `.feature` files before anyone codes the implementation.

---

## Recipe 706: Living Documentation (Adzic)

### Idea
Adzic's most consequential framing: feature files are not tests, they are **living documentation**. They describe what the system does in the present tense. When a behaviour changes, you change the scenario; the documentation updates automatically.

A wiki page about how the coupon system works will drift from reality within months. A `coupons.feature` file that runs on every commit cannot drift -- if it drifts, CI fails.

### Webship-js translation
Every team has the wiki page. "How coupons work", last edited fourteen months ago, three renames behind the UI, still the first search hit for new hires. Documentation that nothing executes is documentation that nothing corrects.

Publish your `.feature` files where stakeholders can read them. Many teams render Gherkin features into a static site (Cucumber's reporting plugins do this). The result: a product specification that's also the test suite that's also the regression guard. One source of truth, three audiences.

Webship-js ships the publishing half: `bin/generate-reports.js` turns each run's JSON output into an HTML report -- and a PDF when you need an artefact to attach to a release ticket -- stamped with metadata like `--app-version` and `--env`. A report generated on every build is living documentation with a timestamp: this is what the system did, on this version, in this environment, at this hour.

**At scale:** publish every project's report to one shared location and you have answered the hardest portfolio question -- "is release X safe?" -- with the same artefact for every product. A manager who cannot read Gherkin can read a green report with the app version in its header. Twenty wiki pages drift in twenty directions; twenty generated reports cannot.

### Recipe
Treat feature files with the same review rigour as production code. Pull-request reviews for `.feature` files should look at language, clarity, completeness -- the same way you'd review docs.

---

## Recipe 707: The Imperative-Declarative Test (North; Cucumber Book)

### Idea
An *imperative* test reads as a list of clicks: "click the login button, type the username, type the password, click submit." A *declarative* test reads as a behaviour: "the user logs in successfully." The Cucumber book recommends declarative scenarios with imperative custom-step implementations.

### Webship-js translation
Watch a new hire's first PR. They need a logged-in user, find a feature file that does it click-by-click, and copy all five steps into their scenario -- the fifteenth copy in the repo. Six months later the login form changes and someone edits fifteen files. That is the cost of imperative scenarios: every detail repeats everywhere it is needed.

```gherkin
# Imperative (avoid in scenarios)
Scenario: User logs in
  Given I am on "/user/login"
  When I fill in "username" with "alice"
  And I fill in "password" with "secret"
  And I click "Submit"
  Then I should see "Welcome"

# Declarative (prefer)
Scenario: User logs in
  When I log in as "alice"
  Then I should see "Welcome"
```

The declarative version reads like a spec. The imperative click-by-click belongs inside the custom step `When I log in as "X"`.

Webship-js's 411 built-in steps are deliberately imperative -- fill, click, select, check, assert. They are the assembly language. Custom steps are the domain language you compile down to it. The discipline is keeping each tier pure: scenarios say *what* in product words; custom steps say *how* in built-ins.

**At scale:** declarative steps are what make a step library portable across a portfolio. `When I log in as "alice"` reads identically on every product, even when one login is a modal and another is a dedicated page -- the per-project step definition absorbs the difference. Teams that share declarative vocabulary across twenty suites can review each other's scenarios in minutes; teams that share imperative click-lists share nothing.

### Recipe
When a sequence of steps occurs more than three times in your feature files, declarativise it -- extract the imperatives into a named custom step. The scenario reads better; the implementation hides where it should.

---

## Recipe 708: The Cucumber Lineage (Hellesøy; Wynne)

### Idea
Cucumber originated as RSpec's "Story Runner" in 2008 (Aslak Hellesøy). It became `cucumber-ruby`, then `cucumber-jvm`, then `cucumber-js`. Each port preserved the Gherkin syntax but adapted to its host language. The Cucumber Book (Wynne, Hellesøy, 2012; second edition 2017) codified the practice: features as documentation, steps as code, tags as configuration.

### Webship-js translation
Webship-js builds on cucumber-js v10 -- the latest JavaScript port. Most of *The Cucumber Book*'s advice carries over verbatim: write declarative scenarios, share setup via Background, parameterise with Scenario Outlines, use tags for environment-aware runs.

The lineage is worth citing when someone on your team calls Gherkin "the new thing we're trying". The syntax has been earning its keep in production suites since 2008. The tools underneath kept changing -- Ruby to JVM to JavaScript, Selenium to Playwright -- but a feature file written in 2010 still reads exactly like one written today. That tells you which part was the durable idea: the language, not the runner.

### Recipe
Read *The Cucumber Book* if you've never read it. Most of webship-js's patterns are direct descendants. The differences are tactical (web-first matchers, smart-waits, AI integration); the strategy is the same as it was in 2012.

---

## Recipe 709: Walking Skeleton (Freeman & Pryce, 2009)

### Idea
*Growing Object-Oriented Software, Guided by Tests* (GOOS) opens with the **walking skeleton**: the smallest possible end-to-end implementation that exercises every layer of the system. It's not feature-complete; it just shows that the layers talk to each other. The walking skeleton is the platform on which subsequent tests are built.

### Webship-js translation
The alternative is the project where tests "will come later". Later arrives in week six; the first scenario fails for five unrelated infrastructure reasons at once -- missing browser dependencies in CI, wrong base URL, a report directory that isn't writable -- and the team concludes acceptance testing is flaky. It was not flaky. It was six weeks of unverified assumptions cashing in on the same afternoon.

For a new project, the walking skeleton is one passing scenario:

```gherkin
Feature: Walking skeleton

  Scenario: Homepage loads
    Given I am on "/"
    Then I should see "Welcome"
```

It tests almost nothing. But it proves: the test framework runs, the fixture server runs, the browser launches, the CI pipeline executes, the report uploads. Every other test builds on that foundation.

In webship-js terms, that one scenario proves more than it reads. The browser launched with your `BROWSER` and `HEADLESS` settings. `LAUNCH_URL` points at a server that actually answers. The smart-wait init scripts installed cleanly. The JSON report landed in `tests/reports/`. The CI runner has every system dependency Playwright needs. Each of those fails loudly and cheaply now -- or mysteriously and expensively later, underneath your fiftieth scenario.

**At scale:** turn the skeleton into a template repository. A new project means clone, set `LAUNCH_URL`, pick the matching pipeline from the thirteen CI configs webship-js ships (GitHub Actions through Bamboo), and be green in CI inside an hour. Track that hour -- time-to-first-green is the most honest onboarding metric a multi-project team has. When project number seven takes three days, your scaffold needs work, not your people.

### Recipe
First scenario of a new project: write the walking skeleton, get it green in CI, then everything else.

---

## Recipe 710: Outside-In vs Inside-Out (GOOS)

### Idea
Two ways to drive TDD: from the outside (start with an acceptance test, work down to units) or from the inside (start with units, work up). GOOS makes the case for **outside-in**: the acceptance test forces you to design the system from the user's perspective, not the developer's.

### Webship-js translation
Inside-out feels productive right up to the end. The API is built, the models are tested -- and then the UI work starts and reveals the flow needs a field the API doesn't return. Outside-in surfaces that on day one, because the scenario describes the screen the user sees, and the screen is the contract.

Webship-js is the acceptance-test layer. Use it to **start** a feature:

1. Write the `.feature` file with the user-visible behaviour.
2. Run it. It fails because no UI exists yet.
3. Drop into your unit test framework. Write the inner units to support the UI.
4. Build the UI. Re-run the feature. Green.

The acceptance test is the steering wheel. Unit tests are the engine.

One refinement: make sure the scenario fails *for the right reason*. Run it at step 2 and read the failure -- it should be a web-first assertion timing out after its 5-second poll because the element doesn't exist yet, not a connection refusal or a broken fixture. A red you understand is a steering wheel. A red you don't is just noise.

### Recipe
For every new feature, write the webship-js scenario first. Use it as the "definition of done." The feature isn't done until the scenario is green.

---

## Recipe 711: Test Doubles -- Dummy, Fake, Stub, Spy, Mock (Meszaros; Fowler)

### Idea
Gerard Meszaros's *xUnit Test Patterns* (2007) and Martin Fowler's "Mocks Aren't Stubs" (2007) named the family:

- **Dummy** -- passed but not used (placeholder)
- **Fake** -- working implementation with shortcuts (in-memory DB)
- **Stub** -- returns canned answers
- **Spy** -- records calls
- **Mock** -- pre-programmed with expectations; fails if expectations aren't met

The five aren't synonyms. Each fits a different testing problem.

### Webship-js translation
Teams blur these words daily -- someone says "mock the API" and means stub it; someone else builds a true mock with expectations and wonders why it breaks on every refactor. The five names matter because each one fails differently when the system changes.

In acceptance tests, you primarily use **stubs** (`the URL "/api/x" returns the JSON: """{"data":"..."}"""`)  and **spies** (`I start recording network requests` + `Then a POST request to "/api/y" should have been made`). Fakes happen at the fixture-server level. Mocks are rare in browser tests -- you usually want stubs and assertions, not pre-programmed expectations.

Under the hood, the stub steps register a `page.route()` interception with glob matching -- a pattern like `**/api/x` catches the call wherever it originates -- and answer from your canned JSON without the request ever leaving the browser. The spy steps record traffic as it happens so you can assert on it afterwards. The same family covers the network conditions themselves: `is blocked`, `is delayed by 2000 ms`, `the network is offline` -- doubles for the wire, not just the responses.

**At scale:** keep canned responses in shared fixture files instead of docstrings duplicated across suites. When the payments API grows a field, you update one fixture set and twenty suites inherit it -- instead of hunting stub payloads through twenty repos. Treat fixture-shape drift like any other dependency upgrade: deliberate, versioned, announced.

### Recipe
Pick the right double for the situation. A stub answers "what does the page receive?" A spy answers "what does the page send?" A mock answers "did the page interact with this dependency exactly this way?" Most acceptance tests need the first two.

---

## Recipe 712: Mockists vs Classicists (Fowler)

### Idea
Fowler distinguished two TDD camps. **Classicists** prefer real collaborators when possible; they test outcomes. **Mockists** mock every collaborator; they test interactions. Each style produces different tests and different designs.

### Webship-js translation
Acceptance tests are inherently *classical*: they use the real browser, the real DOM, often the real backend. Webship-js encourages the classical approach -- mock only what you must (slow APIs, third-party services), keep the rest real.

Every `page.route()` stub you add converts a fact into an assumption -- the test now proves the page works against your guess of the API, not against the API. Sometimes that trade is right: third-party services you can't control, error paths you can't trigger on demand. Make it consciously, one route at a time, instead of mocking the whole backend because it was slow one Tuesday.

### Recipe
Don't over-mock acceptance tests. Each mock is a fidelity loss. If your test passes against mocks and fails against the real system, the test wasn't worth running. Prefer a real (or fake) backend over a fully-mocked one for acceptance.

---

## Recipe 713: The Test Pyramid (Cohn, 2009)

### Idea
Mike Cohn's *Succeeding with Agile* (2009) introduced the **test pyramid**: many fast unit tests at the base, fewer integration tests in the middle, a small number of slow end-to-end tests at the top. The shape is economic -- end-to-end tests are slow and brittle, so you want only as many as necessary.

### Webship-js translation
Acceptance tests sit at the top of the pyramid. They're slow (seconds, not milliseconds) and broad (one scenario covers many layers). You don't want hundreds of them per feature -- you want the **right** dozen.

The economics compound in CI. A unit test costs milliseconds on one process. An acceptance scenario costs a browser, seconds of wall-clock, and runner minutes someone pays for. The pyramid was never an aesthetic -- it is a budget allocation.

**At scale:** multiply by the portfolio. Forty suites running nightly at thirty minutes each is twenty hours of runner time, every night. The pyramid question stops being per-feature and becomes per-portfolio: which scenarios earn an every-commit run, which run nightly, which run weekly before a release? Tag tiers -- `@smoke` versus the full set -- are the pyramid expressed in Cucumber.

### Recipe
For every feature, ask: how few acceptance scenarios cover the behaviour? Three to five is usually right. More than ten is a smell -- you're probably testing implementation details that belong in unit tests.

---

## Recipe 714: The Testing Trophy (Dodds, 2018)

### Idea
Kent C. Dodds proposed the **testing trophy** as a modern revision of the pyramid: more integration tests, fewer pure units. The argument: unit tests on highly mocked code prove little; integration tests catch real bugs.

### Webship-js translation
The trophy thinking pushes you to write more acceptance tests than the strict pyramid would suggest -- because acceptance tests **are** integration tests. They exercise multiple layers in a real browser. What made that uneconomical for a decade was the flake tax, and that is exactly the tax webship-js's machinery removes: `smartSettle` waits for real quiet (pending requests at zero, pending timers at zero, DOM stable) instead of sleeping, and web-first assertions retry every 100ms for up to 5 seconds instead of failing on first look. The trophy is only viable when an extra test costs minutes to write and almost nothing to maintain. That is the bar the tooling has to clear.

### Recipe
Don't dogmatically follow the pyramid. If your unit tests are mostly testing your mocking framework, you need fewer of them and more acceptance tests. The webship-js auto-wait keeps the trophy economically viable.

---

## Recipe 715: The Honeycomb (Schaffer, 2018)

### Idea
Andre Schaffer's **honeycomb** is yet another shape: a small core of integration tests, with a thin shell of unit tests around individual modules and a thin top of end-to-end tests. Used at Spotify and similar microservices shops.

### Webship-js translation
For a microservices architecture, your webship-js suite tests the user-facing surface only; service-level integration tests live in each service's own repo; pure unit tests cover algorithmic complexity. The shape is per-team and per-architecture, not universal.

If your guild meetings have devolved into pyramid-versus-trophy-versus-honeycomb debates, the shape has become a proxy war. The useful questions are concrete: where did the last five escaped bugs come from? Which layer would have caught them? Answer those and the shape draws itself.

### Recipe
Don't pick a shape because a famous engineer wrote about it. Pick the shape that matches your architecture. The webship-js layer is the same in all three shapes -- it's the top of the user-facing pile.

---

## Recipe 716: Test as Specification (Beck; Adzic)

### Idea
Both TDD and SbE treat the test as the spec. The test is not a check **after** development; it is the description **before** development. The implementation must satisfy the test.

### Webship-js translation
You can tell which kind your suite is from the git history. When `.feature` files land in the same pull request as the implementation -- or a sprint later, in a ticket called "add tests" -- they are regression tests. Useful, but they only ever confirm what got built. When the `.feature` file lands first and the implementation PR is the thing that turns it green, the scenario did design work: it decided what done looks like before anyone spent three days building the wrong thing.

Webship-js makes spec-first cheap because its steps bind to what the user sees, not to what the developer builds. `When I fill in "code" with "SAVE10"` resolves the field label-first at runtime -- you can write that line before the form exists, and it starts passing the day a field labelled "code" appears. The spec never has to wait for the implementation to become writable.

### Recipe
Try writing the next scenario before its feature exists. Run it. Watch it fail. Build until it passes. The discipline is uncomfortable for a week; valuable for the rest of your career.

---

## Recipe 717: Refactor Tests Too (Beck)

### Idea
The test code is code. It deserves the same refactoring care as production code. Duplicated assertions, deep nesting, magic strings -- all the smells of production code show up in test code too.

### Webship-js translation
Nobody schedules the decay. It arrives one copied step at a time, until the suite that took minutes to extend takes an afternoon -- and the person extending it is a new hire who copies the worst example in the repo because it was the first grep hit. Test code rots exactly the way production code rots, for exactly the same reason: it gets edited under deadline and cleaned never.

Refactor feature files:

- Extract repeated 3+ step blocks into custom steps (Chapter 50)
- Use `Background:` for shared setup within a feature
- Use Scenario Outlines for repeated structure with varying data
- Name custom steps with domain language (Recipe 687)

Refactor step definitions too:

- Extract repeated logic into helper functions
- Move fixture data into a `fixtures/` directory
- Split monolithic `custom.js` into per-domain files

**At scale:** refactoring twenty suites one at a time never happens -- there is always a release closer than the cleanup. What works is refactoring the shared layer instead: move the duplicated step definitions into one package every suite imports, and a single refactor pays out across the whole portfolio at once. Each project keeps only its scenarios and a thin custom-step file -- the part that genuinely differs.

### Recipe
Schedule a test-suite refactor sprint at least once a quarter. Cost: 2-3 engineer days. Value: orders of magnitude in faster feature delivery and lower maintenance.

---

## Recipe 718: Tell, Don't Ask (in Tests) (Hunt & Thomas)

### Idea
The Pragmatic Programmer's "tell, don't ask" principle applies to test setup. Instead of:

```js
const cart = new Cart();
cart.setItems([...]);
cart.setDiscount(0.10);
cart.setShipping('express');
```

Write:

```js
const cart = aCart().withItems([...]).withDiscount(0.10).withExpressShipping().build();
```

Builders, factories, and DSLs make test setup readable.

### Webship-js translation
Open your worst feature file and find the scenario with eleven setup steps before the one step that actually matters. Whoever reviews it has to run a browser in their head just to locate the behaviour under test -- and whoever copies it (someone will) propagates all eleven lines. A custom step that wraps complex setup is the moral equivalent of Hunt and Thomas's builder:

```gherkin
Given a cart with 3 items, a 10% discount, and express shipping
```

The implementation builds the cart; the scenario reads the intent. Inside that implementation you choose the fastest honest route to the state -- drive the real UI when the journey itself is what you are proving, or seed the cart through the API when it is only scaffolding for the assertion that follows. The scenario reads identically either way, which means you can swap a slow UI setup for a fast API one later without touching a single feature file.

**At scale:** builder sentences are how setup vocabulary stays portable. `Given a cart with 3 items, a 10% discount, and express shipping` can mean a different commerce stack on every project -- the sentence travels, the implementation stays local. Keep the grammar consistent across suites and reviewers stop needing project-specific knowledge just to read setup.

### Recipe
For any test that needs more than 5 lines of setup, build a factory or a custom step. Setup that takes a screen of scrolling is impossible to read; setup that's one declarative sentence is.

---

## Recipe 719: The Saff Squeeze (Beck; Saff)

### Idea
When a failing test produces a confusing error, **inline progressively** until the failure is in a single line. Beck described David Saff's technique: copy the failing assertion to a new test; replace one variable at a time with its concrete value; rerun. Eventually the test fails in a way that names the bug.

### Webship-js translation
The failure that calls for this is never the clear one. It is the scenario that dies on step nine, `When I complete checkout`, with a timeout -- and `I complete checkout` is a custom step wrapping eleven built-ins wrapping a helper someone wrote two years ago. The error names a symptom four layers above its cause.

For a confusing acceptance failure:

1. Copy the failing scenario.
2. Replace abstract steps with their concrete equivalents one at a time.
3. Re-run after each substitution. The point where it starts failing tells you the layer of the bug.

Webship-js sharpens the squeeze. Tag the scratch copy `@screenshots` and you get a capture after every step, not just the failure -- a flip-book of exactly where reality left the script. Drop `Then print current URL` between steps as you inline; it is the cheapest probe there is. And because the copy is disposable, you can be as crude as you like -- it gets deleted the moment it has named the bug.

### Recipe
When a scenario fails mysteriously, don't add console.logs. Saff-squeeze it. Inline one step at a time until the failure is unmistakable.

---

## Recipe 720: The Single Responsibility Test (Beck; SOLID)

### Idea
Each test should test one thing. A test that asserts five things produces five possible failure modes; understanding which one fired is extra work.

### Webship-js translation
One scenario, one behaviour. Cucumber stops a scenario at its first failing step, so a five-assertion scenario fails serially: you fix the first break, re-run, and meet the second -- a drip-feed of bad news that turns one debugging session into five, each with a fresh browser launch in between. The scenario name should answer "what behaviour does this prove?" in one sentence. If you can't, split.

### Recipe
Audit your scenarios. Any scenario whose name is "User can do X and also Y" should be two scenarios.

---

## Recipe 721: Property-Based Testing (Hughes; Hypothesis)

### Idea
John Hughes's QuickCheck (1999) introduced property-based testing: instead of writing example-based tests, write *properties* that should always hold, and let the framework generate random inputs to look for counterexamples. Hypothesis (Python) and fast-check (JavaScript) brought the technique to mainstream languages.

### Webship-js translation
Triangulation (Recipe 702) pins a rule with two examples a human chose. The uncomfortable truth: humans choose polite examples -- round numbers, valid dates, ASCII names. The inputs that break systems are the ones nobody would think to type. Acceptance tests are inherently example-based -- the Gherkin syntax is built around examples. But you can layer property tests at the API level:

```js
const fc = require('fast-check');

it('coupon discount is always non-negative', () => {
  fc.assert(fc.property(
    fc.float({ min: 0, max: 1 }),
    fc.float({ min: 0, max: 10000 }),
    (rate, cart) => {
      const discount = calculateDiscount(rate, cart);
      return discount >= 0;
    }
  ));
});
```

The property runs hundreds of random inputs. The acceptance scenarios cover the canonical examples. The split is economic as much as philosophical: hundreds of runs through a real browser is hours, while hundreds of runs against a pure function is a second -- put the randomness where iterations are free. It is also about audience: a Gherkin example is a communication artefact stakeholders chose on purpose, and a random `{rate: 0.6631, cart: 8841.07}` communicates nothing to anyone.

### Recipe
Use property tests at the layer below acceptance. Use example-based acceptance tests for the user-facing surface. Don't try to do property testing in Gherkin -- it's the wrong layer.

---

## Recipe 722: Mutation Testing (Osherove)

### Idea
Roy Osherove's *The Art of Unit Testing* discusses **mutation testing**: a tool deliberately injects bugs into your code (changing `>` to `<`, removing `!`, etc.) and runs your tests. Bugs that no test catches mean your tests are inadequate -- not just whether they pass, but whether they would notice a bug.

### Webship-js translation
There is a special dread in discovering that a scenario which has passed four hundred runs in a row would also have passed with the feature deleted. Green tells you the assertions held. It never tells you the assertions were worth holding -- that question needs a deliberate experiment.

Mutation testing is mainly for unit-test suites. For acceptance, the moral equivalent is "what happens if I delete this assertion?" If no other scenario catches the gap, the assertion is unique-value and worth keeping. If many other scenarios catch the gap, the assertion is redundant.

**At scale:** a delete-one-assertion pass across forty suites will never happen in one heroic week -- rotate it. One suite per sprint, starting with the scenarios that guard each product's checkout-equivalent. Keep the output: a short list per suite of which scenarios are load-bearing and which are scaffolding. That list is also your priority order the next time the nightly run is over budget and something has to move to weekly.

### Recipe
Once a year, do a "delete one assertion" pass across your suite. Anything that, when removed, leaves no test catching the regression is unique-value. Anything that, when removed, leaves five other tests catching the regression is fat.

---

## Recipe 723: Tests as a Conversation (Adzic; Liz Keogh)

### Idea
Liz Keogh argues a BDD scenario is the *artefact* of a conversation. The conversation matters more than the artefact. The Three Amigos meeting is where shared understanding happens; the scenarios are notes from that meeting.

### Webship-js translation
A scenario written without the conversation is half-value. The same words can mean different things to different stakeholders -- "discount applies to the order" means before shipping to the developer and after shipping to the finance manager, and both read the same Gherkin line and nod. The conversation forces the ambiguity into the open while it is still cheap to fix; the suite stays green either way, which is exactly why the suite alone cannot catch it.

### Recipe
Don't skip Three Amigos to "save time." The 30 minutes you save become 5 hours of rework when the developer's interpretation differs from product's interpretation. Run the meeting. Write the scenarios in the meeting.

---

## Recipe 724: Examples as Tests, Tests as Documentation (Adzic)

### Idea
Adzic's circular insight: examples drive development (they're tests), and the same examples document the system (they're docs). The same artefact serves both. The implication: write examples that read well as documentation, not just as tests.

### Webship-js translation
The honest test takes one minute: ask a product manager to read a scenario aloud. If their eyes glide over `Then the JSON response should have "/data/discount" equal to 1000`, your documentation has an audience of one team -- the team that needs it least, because they wrote it. Re-read your scenarios from a non-engineer's perspective. If a product manager couldn't read the scenario and understand what the system does, the scenario is too implementation-laden.

**At scale:** the reading session doubles as a portfolio consistency check. When one product's scenarios say "the customer" and another's say "the end-user entity", everyone who moves between projects -- QA leads, PMs, support -- pays a translation tax on every visit. Cross-team reading sessions flush out the dialects, and the generated HTML reports give non-engineers a place to do the reading without cloning a repo.

### Recipe
Hold a regular "scenario reading session" with product and design. They read the scenarios aloud. Anywhere they get confused or have to ask "what does this mean?", that's a scenario to refactor.

---

## Recipe 725: The Walking-Backwards Mistake (Adzic)

### Idea
A common mistake: write the implementation, then write the scenarios that match what the implementation does. The scenarios become a tautology -- they pass by definition. Adzic calls this the "walking backwards" failure mode.

### Webship-js translation
You can usually date the moment a suite started walking backwards: it is the sprint when "write the tests" became a separate ticket, scheduled after the feature ticket, often assigned to a different person. Whatever that ticket produces will describe the implementation, quirks included -- because the implementation was the only specification its author had. Write scenarios **before** implementation, or at least independently of it. If you find yourself writing a scenario to fit an existing endpoint's quirks, stop -- the quirks are probably bugs, and you are about to give them a green checkmark that makes them permanent.

### Recipe
A test that has never been red is a test you don't trust. New scenarios should fail first.

---

## Recipe 726: The Coverage Trap (Beck)

### Idea
High test coverage is necessary but not sufficient. A suite with 95% coverage and weak assertions still ships bugs. Coverage measures lines executed, not behaviour verified.

### Webship-js translation
The number arrives in a slide deck: "95% coverage." Two sprints later a customer finds the bug the deck said could not exist, and the awkward retro rediscovers what coverage always measured -- that lines ran, not that anyone checked the result of running them.

You can't measure acceptance-test coverage in line-counts. The right metric is **behaviour coverage**: are all the documented user journeys exercised? Are all the documented edge cases exercised? Coverage tools won't tell you. You and your team will, by reading the scenarios.

**At scale:** behaviour coverage is also the only coverage a PMO can compare across products. Line coverage of a CMS site and a React SPA measure different universes; "every documented checkout journey has a green scenario" means the same thing on both. Keep the journey list next to each suite and report against it -- it turns "is release X safe?" from a feeling into a checklist anyone can audit.

### Recipe
Don't chase a line-coverage number for acceptance tests. Track behavioural coverage by listing user journeys and matching them to scenarios. A journey without a scenario is a gap; a scenario without a journey is fat.

---

## Recipe 727: Tests as a Safety Net (Beck)

### Idea
The deepest reason to write tests is courage. Tests let you change code without fear. A suite that catches regressions in 30 seconds turns "should we refactor?" into "yes, let's go." A suite that takes 4 hours to run does the opposite.

### Webship-js translation
Watch what happens when someone proposes replacing the auth module. On the team with the four-hour suite, the room goes quiet and the refactor dies in the backlog. On the team with the fifteen-minute suite, somebody says "the scenarios will catch it" and the work starts Monday. Same risk, different courage -- and the only variable is the feedback loop.

A fast, reliable webship-js suite is courage. It says "yes, let's rip out the old auth system" because you'll know in 5 minutes if anything important broke.

**At scale:** courage is set by the slowest suite a change must clear. If a platform upgrade touches twenty sites and three of them have 90-minute suites, those three set the pace for all twenty -- so the upgrade gets batched, postponed, and riskier. Put wall-clock time per suite on one dashboard and treat a suite drifting past budget as a defect to fix, not a fact of life.

### Recipe
Track your suite's wall-clock time. Anything over 15 minutes for the full suite breaks the courage loop. Shard, parallelise, or split into smoke-and-full tiers (Chapter 47, Recipe 532) to keep iteration tight.

---

## Recipe 728: Don't Test the Framework (Fowler; Kent Beck)

### Idea
A common waste: writing tests that verify the framework's own behaviour rather than your code. If Cucumber's `Given X` step works, you don't need a test that asserts `Given X` worked -- the framework's tests already do.

### Webship-js translation
These scenarios accumulate innocently. Someone wants a "quick sanity check", writes a scenario that navigates and then asserts the navigation happened, and it passes forever -- because what it actually proves is that Playwright works. Three years later there are forty of them, each burning a browser session per run and guarding nothing the framework's own test suite doesn't already guarantee.

```gherkin
# Wasted test -- verifies Playwright/Cucumber, not your app
Scenario: I can navigate to the homepage
  Given I am on "/"
  Then the url should match "/"

# Real test -- verifies your app's contract
Scenario: Homepage shows the welcome message
  Given I am on "/"
  Then I should see "Welcome to Webship.co"
```

**At scale:** dead-weight scenarios are a CI line item. One asserts nothing app-specific yet still costs a browser launch, a settle cycle, and runner seconds -- multiplied by forty suites and 365 nights. A yearly hunt for assertions that would pass on any page is the cheapest CI cost cut available, because deleting them loses exactly nothing.

### Recipe
Re-read every scenario. If it would pass against any random page, it's testing the framework. Replace with a real assertion.

---

## Recipe 729: The Right Failure Message (Beck)

### Idea
A failing test should tell you what's wrong in one sentence. "Expected true but got false" is useless. "Expected the order total to be $100 but got $90" is actionable. Beck spent years iterating xUnit's failure messages because the message **is** the test's interface to a debugging engineer.

### Webship-js translation
The message gets read at the worst possible moment -- 7:40am, twenty red scenarios, release at noon, by someone who did not write the test. "Assertion failed" at that moment is not a message; it is homework.

Webship-js's friendly-error pipeline (Chapter 44) is the same idea applied to acceptance tests. The built-in assertions already speak in specifics -- expected versus actual, selector and timeout named -- and every failure arrives with its screenshot (capture-on-failure is on by default) plus whatever console errors the JS-error hooks collected. Your custom steps are the one place vagueness can creep back in. Hold them to the same standard:

```js
throw friendly(`Expected the order total to be $${expected} but got $${actual}. The page is ${url}, the cart contains ${itemCount} items.`);
```

**At scale:** failure messages are the interface between your projects and whoever is on triage rota. When the person reading the morning's failures supports twelve products, a message that names the expected value, the actual value, and the URL is the difference between thirty seconds and thirty minutes per red scenario. Write every message for a reader who has never seen the project -- on a multi-project team, that reader is the normal case.

### Recipe
Audit your custom-step error messages once a quarter. Anywhere the message is "Assertion failed", rewrite to include the values being compared and the context the user would need.

---

## Recipe 730: One Failure, One Diagnosis (Beck)

### Idea
When a single test failure produces multiple stack traces, the cause is multiple. Aim for tests where **one failure means one cause**. The classical TDD discipline -- one assertion per test, when reasonable -- enforces this.

### Webship-js translation
Broad scenarios feel efficient on the day you write them -- one browser session, ten assertions, look at all that coverage. They bill you later, on the morning one goes red and the name "Checkout works correctly" tells you nothing about which of its ten promises broke. A scenario with 10 `Then` steps will produce 10 possible failures. The first to fail stops the rest -- so a single red run shows you one break and hides the other nine behind it. Prefer narrow scenarios over broad ones.

### Recipe
When a scenario has more than 5 `Then`s, look for natural splits. Two narrower scenarios beat one wide one for diagnosability.

---

## Recipe 731: The First Test Hardest (Beck)

### Idea
Writing the first test for a new codebase is the hardest because you have to decide on the fixture, the harness, the build, the CI. Once it's in place, the second test takes 10% of the effort. Get the first one in early, even if it's almost-trivial.

### Webship-js translation
Teams postpone the first test waiting to feel ready -- the naming-convention debate, the folder-structure debate, the should-we-use-tags debate. Beck's point cuts through all of it: the first test's job is to exist, because it forces every infrastructure decision (fixture, browser, CI, report) to be made once, cheaply, while nothing depends on them yet. The walking skeleton (Recipe 709) is the first test. Get it green in CI before you've decided on naming conventions, before you've written your style guide, before you've designed your test World. Everything else follows.

### Recipe
First commit on a new project should include a passing `.feature` file. Even if it's `Scenario: Homepage exists`. The cost is 10 minutes; the value is unlocking every subsequent test.

---

## Recipe 732: Tests Are an Honest Record (Beck)

### Idea
If a test passes, the system has the behaviour the test describes. If a test fails, it doesn't. There is no in-between. The test is honest in a way intuition isn't. "I'm pretty sure the login works" is a feeling. "The login scenario is green" is a fact.

### Webship-js translation
This cuts both ways, and the painful direction is the second one. "I'm sure that still works" has shipped more regressions than any other sentence in software -- and its mirror, "that test always fails, ignore it", is how a team trains itself to stop reading its own record. Believe the suite. If a scenario says the system does X and you think it doesn't, either the scenario is wrong (refactor) or your belief is wrong (the system actually does X). Don't override the suite with intuition -- and don't leave a scenario in the suite once you've decided its word no longer counts.

### Recipe
When a scenario contradicts your intuition, investigate the scenario, not your intuition. Often the scenario is right and you've been operating on stale beliefs.

---

## Recipe 733: Refactoring Without Tests Is Editing (Feathers; Fowler)

### Idea
Michael Feathers's *Working Effectively with Legacy Code* defines legacy code as "code without tests." Refactoring legacy code without first writing characterisation tests isn't refactoring -- it's editing with hope.

### Webship-js translation
The scary version is the rewrite nobody can verify. The old checkout "just works", the person who built it left in 2023, and now you are replacing the framework underneath it. Without tests, every behaviour you didn't know existed is a behaviour you are about to delete -- and you will learn the full list from support tickets.

Before refactoring a UI feature, write the acceptance scenario for the **current** behaviour. Run it. Green. Then refactor. The scenario tells you when you've broken something.

**At scale:** characterisation suites are how a platform team survives portfolio-wide upgrades. When we moved Varbase through major Drupal upgrades at Vardot, the acceptance suite was the characterisation suite -- the same scenarios ran against the old major and the new one, and every difference was either a planned change or a regression caught before a single site upgraded. One suite, written once, de-risked the same migration for every site built on the distribution.

### Recipe
For any feature you're about to rewrite, write the acceptance scenarios first as a characterisation suite. Spend a day on it. The day pays back tenfold during the rewrite.

---

## Recipe 734: The Test Coverage / Behaviour Coverage Distinction (Adzic)

### Idea
Two different coverage measures:

- **Test coverage** -- what lines of code execute
- **Behaviour coverage** -- what user behaviours are exercised

A 95%-test-coverage suite that hits 30% behaviour coverage is dangerous. The reverse -- 60% test coverage but 95% behaviour coverage -- is much safer.

### Webship-js translation
The two numbers answer different questions from different people. Test coverage answers the engineer's question -- did my code run under test? Behaviour coverage answers the product owner's -- can a customer actually do the things we sold? A release decision made on the first number alone is a decision made about the wrong question. Webship-js scenarios measure behaviour coverage. Track them by category:

- Happy paths covered? (yes / no)
- Common error paths covered?
- Edge cases (zero, negative, max) covered?
- Cross-locale / cross-device covered?

### Recipe
Maintain a behavioural-coverage checklist alongside your test suite. The checklist tells you what's tested and -- more importantly -- what's not.

---

## Recipe 735: BDD is About Communication (North; Adzic; Keogh)

### Idea
BDD's core idea is not "tools" or "syntax" -- it's **communication**. Scenarios are a contract among engineering, product, and QA. The shared understanding is the value; the executable test is a happy side effect.

### Webship-js translation
The tooling makes this easy to forget. A team can install cucumber-js, write five hundred scenarios, keep them green in CI, and still practice zero BDD -- because nobody outside engineering ever read one. Treat scenarios as the contract. If product reads a scenario and disagrees, that's the highest-value moment in the whole delivery cycle to catch a misunderstanding -- the code does not exist yet, so the fix costs an edit to a text file instead of a sprint.

### Recipe
Add "review the scenarios" as an explicit step in your product workflow. Product manager signs off on the scenarios; engineering builds against them.

---

## Recipe 736: Don't Skip the Refactor (Beck)

### Idea
The "refactor" step in red-green-refactor is the one teams skip first. The temptation: once the test is green, move on. The cost: technical debt compounds, tests become hard to read, the suite gets slow.

### Webship-js translation
The skip never announces itself. It sounds like "we'll clean this up next sprint", said sincerely, sprint after sprint, until the suite is a hundred scenarios of copy-paste that everyone resents extending. After each new scenario passes, allocate 5-10 minutes to refactor while the context is still in your head:

- Is the scenario name still accurate?
- Are there duplicated steps that should be a `Background:`?
- Are there imperative steps that should be declarative?
- Are there custom steps to extract?

### Recipe
Build "refactor minute" into your team's TDD loop. Some teams use a literal kitchen timer set to 5 minutes after a green light. The discipline pays off.

---

## Recipe 737: Test Names as Sentences (Beck)

### Idea
A good test name reads as an English sentence describing the behaviour. `test_foo_bar_baz_1` is uninformative. `When a coupon code expires, applying it shows the "expired" message` is.

### Webship-js translation
Gherkin scenarios are named sentences by design:

```gherkin
Scenario: Expired coupons show an "expired" message when applied
```

Use scenario names as the index into your suite. A grep through the names should produce a readable product overview. Names are also what every report leads with: the HTML report renders one line per scenario, and a manager scanning it reads names, never steps. "Coupon test 3" wastes that audience; "Expired coupons show an expired message when applied" briefs them.

**At scale:** consistent naming turns grep into a portfolio query. When all twenty suites name scenarios as behaviour sentences, one search across repos produces a readable inventory of what the whole product line actually does -- the cheapest capability audit a PMO will ever get. The convention costs nothing on day one; retrofitting it three years in costs a quarter.

### Recipe
After writing each scenario, re-read just its name. If it reads as a sentence describing a real behaviour, ship. If it reads as a label or a category, rename.

---

## Recipe 738: Beyond Green and Red -- The Pending Test (North; Cucumber Book)

### Idea
A test can be **pending** -- written, but not yet implementing. Useful when you've identified a behaviour during a Three Amigos session but don't have time to implement immediately.

### Webship-js translation
Every good Three Amigos session produces more scenarios than the sprint can absorb, and the overflow has to live somewhere. In most teams it lives in meeting notes -- which is to say, it dies. The fifth time someone says "we said we'd test that" after a production incident is when teams learn to keep the overflow in the repo instead.

Tag scenarios `@pending` and exclude them from the default run with a tag expression (`--tags "not @pending"` in your default profile):

```gherkin
@pending
Scenario: Loyalty points award on first purchase
  Given I am a new customer
  When I place my first order
  Then I should see "You earned 50 loyalty points"
```

The scenario captures the requirement now; implementation happens later.

**At scale:** @pending is backlog evidence a PMO can audit. Counting pending scenarios per project shows where agreed behaviour is outrunning implementation -- a leading indicator no ticket system gives you, because tickets rarely state acceptance criteria in executable form. A project whose pending count only ever grows is holding conversations it cannot cash.

### Recipe
Use `@pending` instead of TODO comments. A scenario tagged @pending is searchable, tracked, and visible in reports as outstanding work.

---

## Recipe 739: The Definition of Done (Adzic; Crispin & Gregory)

### Idea
Lisa Crispin and Janet Gregory's *Agile Testing* defines done as "feature complete AND tests passing AND examples documented." Acceptance scenarios are the documentation. A feature isn't done until its scenarios are green.

### Webship-js translation
Without a definition, "done" decays into "demoed". The feature works in the sprint review, the ticket closes, and three weeks later the edge cases nobody wrote down surface in production -- where edge cases prefer to surface. Adopt the Crispin-Gregory definition in your team's workflow:

- Feature merged → required scenarios present
- Feature deployed → all scenarios green in production environment
- Sprint complete → no `@pending` from this sprint

**At scale:** a shared definition is what makes "done" comparable across a portfolio. When every project's gate is "relevant scenarios green in CI", release readiness review becomes twenty report links instead of twenty status meetings -- stamp each report with `--app-version` and `--env` and the evidence describes itself. The projects that negotiate private definitions of done are the ones that surprise you in the release retrospective.

### Recipe
Make the definition explicit. Put it on the wall, the wiki, the PR template. "DoD: relevant webship-js scenarios are green in CI."

---

## Recipe 740: Acceptance Tests Are Slow on Purpose (Cohn; Fowler)

### Idea
Acceptance tests are slower than unit tests because they exercise more layers. That's not a bug -- it's how they catch integration issues. Trying to speed them up to unit-test levels usually means mocking out the layers they're meant to test.

### Webship-js translation
Sooner or later someone proposes "speeding up the suite" by stubbing the backend, and the proposal sounds like engineering. Recipe 712 named the price: every stub trades a fact for an assumption. The seconds an acceptance scenario spends are not waste -- they are real fetches, real renders, real settles. `smartSettle` waiting for the page's pending requests and timers to reach zero *is* the test; that wait is precisely the integration the unit tests below could never check.

A 5-minute webship-js suite is **fast** for acceptance tests. A 30-second suite is suspect -- did you mock the backend? If so, you may have lost the value.

### Recipe
Optimise acceptance suite speed by parallelisation (Chapter 47, Recipe 551) and selective tagging (smoke vs full). Don't optimise by mocking out the system you're trying to test.

---

## Recipe 741: Failure as Feedback (Beck)

### Idea
A test failure is feedback. The faster the feedback, the more often you act on it. Beck argues for feedback in seconds, not minutes -- because a minute of feedback breaks flow.

### Webship-js translation
Feedback that arrives in seconds changes your next keystroke; feedback that arrives tomorrow changes your calendar. Acceptance tests will never hit Beck's seconds, but the distance between "thirty seconds after save" and "after the nightly run" is the distance between fixing a bug while the code is still in your head and reconstructing the whole context at standup.

Run a tagged smoke subset on every save (with a file-watcher). Run the full suite in CI on every commit. Receive feedback every 30 seconds locally, every 5 minutes in CI.

### Recipe
Cucumber-js has no watch mode of its own, so wrap it in one: `npx nodemon --ext js,feature --watch tests --exec "npx cucumber-js --tags @critical-path"`. Save a file, smoke runs, get told immediately if you broke something. The 30-second feedback loop is the closest acceptance testing gets to TDD's tightness.

---

## Recipe 742: The Test Suite as Product (Adzic)

### Idea
The test suite is a product the engineering team builds, maintains, and uses. It has users (the team), features (which behaviours it covers), bugs (flakes, false negatives), and a roadmap (which areas need more coverage). Treat it like a product.

### Webship-js translation
Suites nobody owns develop the symptoms of any unowned product: features nobody asked for, bugs nobody fixes, users who quietly stop trusting it. The day a developer mutters "just re-run it, it does that sometimes" is the day your product lost a customer.

Track the suite's health metrics:

- Pass rate (target: 100% on `main`)
- Flake rate (target: <1%)
- Wall-clock time (target: <15 minutes)
- Behaviour coverage (target: tracked, growing)
- Custom-step count (target: small, audited)

The raw material is already there: every run writes `tests/reports/cucumber_report.json`, and the report generator stamps each HTML or PDF artefact with `--app-version`, `--env`, `--browser`, and any `--metadata Key=Value` you pass. Pass rate, duration, and flake trends are a small script over JSON you are already producing.

**At scale:** one suite's metrics are a curiosity; forty suites' metrics on one board are an operations tool. Normalise the metadata -- same environment names, same version stamps -- and the board starts answering real questions: which products flake worst, which suites blew their runtime budget, which have not gained a scenario in six months while features kept shipping. That last one is the silent killer: the suite didn't break, it just stopped describing the product.

### Recipe
Add suite-health to your engineering metrics dashboard. The suite is too important to be an afterthought.

---

## Recipe 743: Tests as Designer (Beck)

### Idea
Writing the test first **forces** good design. If your test is painful to write, the code is painful to use. The test is a design tool.

### Webship-js translation
At the acceptance level, a scenario that's painful to write usually means the UI is hard to use. Eight steps for a single user action suggests too many clicks; deeply nested step dependencies suggest tangled UI. The signal is easy to dismiss as test-framework friction -- "webship-js needs too many steps for this" -- when it is actually the interface confessing. If QA needs twelve steps to apply a coupon, so does every customer, and customers don't get a green checkmark for finishing.

### Recipe
Use the scenario's readability as a UI signal. "I had to write 12 steps for this simple action" is feedback for the designer.

---

## Recipe 744: The Discipline of Small Steps (Beck)

### Idea
Beck's *TDD by Example* opens with the smallest possible code change between red and green. Why? Because small steps mean you always know what you just did. If the test goes red after the change, the bug is in the last 30 seconds of code, not in the last 30 minutes.

### Webship-js translation
Implementing a feature scenario? Make the smallest change that moves the scenario closer to green. Run it. Move on. The scenario doubles as a progress bar: each run should fail one step further down than the last, and a failure that jumps *backwards* tells you the latest change broke something earlier -- while that change is still thirty seconds old and trivially revertible.

### Recipe
Resist the urge to "implement everything for this scenario in one go." Each commit should change one thing. The scenario tells you when you're done.

---

## Recipe 745: The Two Heuristics for Test Names (Beck; Fowler)

### Heuristic 1: Behaviour, Not Implementation

**Bad:** `Scenario: GET /api/users returns 200`
**Good:** `Scenario: Anyone can list public user profiles`

### Heuristic 2: Specific, Not Generic

**Bad:** `Scenario: User login`
**Good:** `Scenario: User logs in with valid credentials`
**Better:** `Scenario: User logs in successfully and lands on the dashboard`

### Webship-js translation
Both apply directly to your scenario names. Read each name aloud. If it sounds like a category label, replace with a behaviour sentence.

### Recipe
Hold a scenario-naming review. Read each name. Anything that fails either heuristic, rename.

---

## Recipe 746: The Edge Case Hunt (Cucumber Book)

### Idea
Wynne and Hellesøy recommend a deliberate edge-case hunt during Three Amigos. For each user story, ask:

- What if the input is zero, negative, very large, or empty?
- What if the user is offline, on a slow network, or using an old browser?
- What if two users do this simultaneously?
- What if the operation is repeated?

Each "what if" that produces interesting behaviour becomes a scenario.

### Webship-js translation
Add an edge-case section to your `.feature` files:

```gherkin
Feature: Apply a coupon

  # Happy path
  Scenario: Valid coupon applies
    ...

  # Edge cases
  Scenario: Empty coupon code shows validation error
  Scenario: Coupon applied twice does not double-apply
  Scenario: Expired coupon shows expired message
  Scenario: Coupon larger than cart total caps at cart total
```

### Recipe
For every new feature, write at least three edge-case scenarios alongside the happy path. The edge cases are where the bugs live.

---

## Recipe 747: The Definition of Ready (Crispin & Gregory)

### Idea
A user story is **ready** for development when it has:

- A clear acceptance criteria
- A small enough scope to fit in a sprint
- Concrete examples (your scenarios)

Stories without acceptance scenarios are not ready. They're wish-lists.

### Webship-js translation
Before estimating or scheduling a story, write its acceptance scenarios. If you can't, the story isn't well-enough understood to estimate. Send it back to product for refinement.

### Recipe
Add scenarios as a Definition of Ready in your team's process. The 30 minutes it takes to write three scenarios catches half a sprint's worth of ambiguity.

---

## Recipe 748: The Production Test (Crispin & Gregory)

### Idea
Some tests should run in **production**, against the real live system, to catch incidents the moment they happen. Synthetic monitoring -- a probe that does a real user journey every 5 minutes -- is a form of acceptance test in production.

### Webship-js translation
Tag a subset of scenarios `@production-safe` (read-only, idempotent, low-traffic). Schedule them to run against production every 5-15 minutes. Alert on failure.

### Recipe
Pick three smoke scenarios. Run them against production every 10 minutes. The 5-second alert when checkout breaks at 3am beats the 1-hour delay from waiting for a customer to report it.

---

## Recipe 749: Test Heuristics (James Bach; Lisa Crispin)

### Idea
Software testers maintain heuristics -- rules of thumb for finding bugs. Examples: "test at boundaries", "test for races", "test the error path more than the happy path." Lisa Crispin's *Agile Testing* lists dozens.

### Webship-js translation
For each new feature, run a heuristic checklist:

- Boundary: zero, one, many, max
- Race: two clicks rapid, two sessions concurrent
- Network: slow, offline, intermittent
- Time: midnight rollover, leap second, DST transition
- Locale: RTL language, multi-byte characters, currency
- Failure injection: backend returns 500, 404, timeout

### Recipe
Build a personal or team heuristics checklist. Run it for every feature. The list grows as you encounter new bug-classes.

---

## Recipe 750: The Long Discipline (Beck; Adzic; Wynne)

### Idea
Every author in the canon ends with the same point: testing is a discipline. Not a tool, not a methodology -- a discipline. Like any discipline, the gains compound over years. The team that practiced TDD or BDD for ten years writes faster, cleaner code than the team that decided to "try TDD" for a quarter.

### Webship-js translation
Don't try webship-js for a quarter. Adopt it for a decade. Build the muscle of writing scenarios first. Build the habit of refactoring tests. Build the team norm of Three Amigos. The compounding starts slow and ends powerful.

### Recipe
Pick one practice from this chapter. One. Practice it for a month. Then pick another. After a year, you'll have twelve practices internalised -- and a test suite that compounds value every commit.

---

## Closing Thoughts From the Canon

A short list of quotes worth reading in full -- with the verbatim words from each author's own book:

- **Kent Beck**, *Test-Driven Development by Example* (2002): "Clean code that works." (Quoted in Harry Percival's *Test-Driven Development with Python* as the goal of TDD.)
- **Dan North**, "Introducing BDD" (2006); foreword to *The Cucumber Book* (2012): "BDD has come a long way since I first started talking about it in 2003... I was simply trying to find better ways to explain the revelatory practice of TDD... Over time we were able to take the action closer to the business stakeholders and describe the multilayered approach that is modern BDD (and is also, ironically, classic TDD, which Kent Beck described from the very outset as working on multiple levels of abstraction)."
- **Matt Wynne & Aslak Hellesøy**, *The Cucumber Book* (2012, 2017): "Cucumber is a friendly tool. It wants to be part of your team... Cucumber will volunteer to do all the boring repetitive checks... freeing your testers up to do interesting, creative work instead." And: "Cucumber is more than a testing tool; it's a collaboration tool."
- **Ian Dees, Matt Wynne, Aslak Hellesøy**, *Cucumber Recipes* (2013): "Cucumber is first and foremost a communication tool. It helps you express in clear terms what your software is supposed to do and why."
- **Steve Freeman & Nat Pryce**, *Growing Object-Oriented Software, Guided by Tests* (2009): "Writing a test first forces us to clarify our intentions, and we don't start the next piece of work until we have an unambiguous description of what it should do. The process of writing a test first helps us see when a design is too rigid or unfocused."
- **Robert C. Martin**, foreword to *The RSpec Book* (2010): "The meme is *Craftsmanship*."
- **Harry Percival**, *Test-Driven Development with Python* (2014): "Obey the Testing Goat" -- the discipline of writing tests first, even when deadlines whisper otherwise.
- **Enrique Amodeo**, *Learning Behavior-Driven Development with JavaScript* (2015): "A lot of people see the whole TDD/BDD approach as a black-and-white decision. Either you do not do it, or you try to achieve a hundred percent test coverage. The real world calls for a more pragmatic approach: write the tests that really pay off and do not write those that do not give you much value."
- **Gojko Adzic**, *Specification by Example* (2011) and *Bridging the Communication Gap* (2009): living documentation, three amigos, key examples.
- **Liz Keogh**, BDD talks (2008-present): "BDD is about the conversation, not about the tools."

If this chapter is the only thing you read on BDD theory, you're well-armed. If it's the bibliography you read **before** the books, you'll find familiar ideas wherever you go. Either way, you stand on the shoulders of a generation that worked these patterns out by hand. Build on their work; don't reinvent it.

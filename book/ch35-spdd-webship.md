# Chapter 35: Structured Prompt-Driven Development with Webship-js

---

Here is the thing. AI changed how we write code. It did not change how we write specifications. SPDD -- Structured Prompt-Driven Development -- is the discipline that makes that gap visible and closes it. Coined by Thoughtworks and championed by Martin Fowler, SPDD treats prompts as first-class delivery artifacts: version-controlled, reviewed, reused, and improved over time.

Webship-js fits this discipline naturally. Gherkin scenarios are already structured prompts in disguise. They describe what the system should do, in domain language, with executable tests behind them. Combine SPDD with webship-js, and you get a closed loop: human writes the structured prompt, AI generates code, webship-js verifies behavior, and -- when reality diverges -- you fix the prompt first, then regenerate the code.

This chapter shows the full loop.

<!-- diagram: tdd-ai-loop -->

```
┌────────────────────────────────────────────────────────────────┐
│ Human          →     AI          →     Tests                   │
│ (specifies)          (generates)        (verify)                │
│                                                                 │
│       ↑                                  │                      │
│       └──────── on failure, iterate ─────┘                      │
└────────────────────────────────────────────────────────────────┘
```

---

## What SPDD Actually Means

> **Why this matters:** Vibe coding -- chatting your way to a feature -- works for prototypes. It collapses for production. SPDD restores engineering discipline to prompt-led work.

Thoughtworks coined SPDD after watching teams use LLM assistants in their internal IT organization. The pattern they saw: ad-hoc chats produce code that ships, then no one can recreate the conversation that produced it. Six weeks later the code needs a small change. The team prompts the model again. The model produces a different design. Drift starts.

Now multiply that by a portfolio. Ten products, each with its own undocumented chat history, is an organization whose most important design decisions are written down nowhere at all.

SPDD fixes this by **versioning the prompt itself**, not just the code. The prompt becomes the source of truth. When code changes, the prompt updates. When reality diverges from the prompt, you fix the prompt first.

Three ideas drive the practice:

1. **Prompts are artifacts.** Commit them. Review them. Reuse them.
2. **Structure beats freeform.** A canvas with named sections forces clarity that "write me a login screen" never will.
3. **Closed loop, not one-shot.** Generate -> review -> refine -> regenerate. Never a single ask-and-ship.

---

## The REASONS Canvas

<!-- diagram: reasons-canvas -->

```
┌──────────────────────────────────────────────────────────────────┐
│ R Requirements    Problem & definition of done                   │
│ E Entities        Domain entities and relationships              │
│ A Approach        Strategy to meet the requirements              │
│ S Structure       Components and dependencies                    │
│ ─────────────                                                    │
│ O Operations      Concrete, testable steps (signatures, types)   │
│ ─────────────                                                    │
│ N Norms           Cross-cutting engineering standards            │
│ S Safeguards      Non-negotiable boundaries (security, perf)     │
└──────────────────────────────────────────────────────────────────┘
```

The seven sections move from abstract intent ("why are we doing this") to concrete execution ("what must we not do"). The first four are abstract and design-focused. **Operations** is the bridge -- it pins down concrete signatures and steps. The last two govern: **Norms** for engineering standards, **Safeguards** for invariants the system must never violate.

The order is also a review router. Product owners can sign off on Requirements and Entities without reading code. Architects argue about Approach and Structure. QA owns Operations. Security gets a named place to put non-negotiables instead of a comment thread that scrolls away. One canvas, four audiences, no translation step.

> **Note:** When the LLM generates code from a complete REASONS canvas, the output stays inside the boundary you defined. Hallucinations drop. Reviews speed up.

---

## How Webship-js Maps to REASONS

Webship-js artifacts already cover most of the canvas. Here is the mapping:

| REASONS section | Webship-js artifact                                            |
|------------------|----------------------------------------------------------------|
| Requirements     | `Feature:` description (As / I want / So that)                 |
| Entities         | Domain nouns referenced in step text                           |
| Approach         | Background section + scenario outlines                         |
| Structure        | `tests/` layout + `cucumber.js` `require:` + custom step files |
| Operations       | Individual `Given / When / Then` steps                         |
| Norms            | `worldParameters` + project conventions                        |
| Safeguards       | Tags (`@critical`, `@security`), assertions, `Then` invariants |

A well-written `.feature` file is a structured prompt. The novelty SPDD adds is the **discipline** of treating it like one -- versioning it, reviewing it, never letting it drift from the code it specifies.

Two rows deserve a closer look. Norms map to `worldParameters` because that is where project-wide policy already lives in webship-js -- the `launchUrl` each environment overrides with the `LAUNCH_URL` env var, the selector registry every scenario inherits, the breakpoint map your responsive scenarios share. And Safeguards map to tags because tags are *executable* policy: `@critical` decides what blocks a merge, and a `@js-fail` tag turns any JavaScript console error into a hard failure for that scenario. A safeguard written as a tag gets re-checked on every run. A safeguard written in a wiki gets re-checked never.

---

## Recipe 336: Writing a Structured Prompt as a Webship-js Feature

### Problem
Sprint planning. The product owner drops "users should be able to reset their password" into the backlog and moves on. QA writes scenarios from guesswork, the developer makes different guesses in code, and three weeks later security asks why there is no rate limit -- a requirement that existed only in someone's head. The story was never wrong; it was never *structured*. You want to turn it into a prompt precise enough that an AI -- or a new teammate -- can build from it, and webship-js can verify it.

### Solution

Apply the REASONS canvas inside the feature file:

```gherkin
Feature: Password reset

  # R - Requirements
  # As a registered user
  # I want to reset my password via email
  # So that I can regain access if I forget it
  # Definition of done:
  #   - email arrives within 60 seconds
  #   - token expires after 24 hours
  #   - reset succeeds; old password no longer works

  # E - Entities
  # User (email, password_hash, reset_token, reset_token_expires_at)
  # ResetEmail (to, subject, body, token_link)

  # A - Approach
  # Use a single-use signed token. Email it. Validate on submit.

  # S - Structure
  # Pages: /forgot-password, /reset-password?token=...
  # API: POST /api/auth/forgot, POST /api/auth/reset

  # O - Operations (these become the scenarios below)
  # N - Norms
  # All copy in English + Arabic. WCAG AA. Rate-limited 3/hour.
  # S - Safeguards
  # Token never logged. Rate limit enforced. Old session invalidated.

  Background:
    Given I am on "/forgot-password"

  @critical @auth
  Scenario: Request reset email
    When I fill in "Email" with "user@example.com"
    And I press "Send reset link"
    And I wait for AJAX to finish
    Then I should see "Check your inbox"
    And the response status code should be 200

  @critical @auth @security
  Scenario: Token expires after 24 hours
    Given a reset token was issued 25 hours ago for "user@example.com"
    When I am on "/reset-password?token=expired-token"
    Then I should see "This link has expired"
    And I should not see "New password"

  @critical @auth @security
  Scenario: Old password stops working after reset
    Given I have a valid reset token for "user@example.com"
    When I am on "/reset-password?token=valid-token"
    And I fill in "New password" with "NewSecure123!"
    And I press "Reset password"
    And I wait for AJAX to finish
    Then I should see "Password updated"
    When I am on "/login"
    And I fill in "Email" with "user@example.com"
    And I fill in "Password" with "OldPassword456"
    And I press "Sign in"
    Then I should see "Invalid credentials"
```

The comment block at the top is the REASONS canvas. The scenarios are the **Operations** -- the concrete, testable, signature-precise steps. Norms (Arabic copy, WCAG AA, rate limit) and Safeguards (token never logged, rate limit, session invalidation) are encoded as additional scenarios or tags that downstream steps verify.

### Why this works
The whole document is committed to git. Reviewers see intent (R), domain (E), strategy (A), boundaries (N + S), and verification (Operations) in one place. When an engineer six months later asks "why does the reset flow rate-limit?", the answer is right there in the prompt that drove the implementation.

Look closely at the Operations layer, because it mixes two kinds of steps on purpose. `I fill in "Email" with ...`, `I press ...`, `I should see ...`, and `the response status code should be 200` are built-in webship-js steps -- the fill step resolves the field label-first, and the AJAX wait hands control to smartSettle, which watches the page's instrumented fetch/XHR counter and pending timers until both hit zero and the DOM goes quiet. But `Given a reset token was issued 25 hours ago for ...` is a domain step that does not exist yet. That is not a mistake -- it is the canvas doing its job. The Operations section pins down the *signature* of the custom step before anyone implements it, so the step definition becomes part of the work the prompt specifies. Notice also that the expired-token scenario asserts the absence of the "New password" field rather than "the form is gone" -- assert on what a user would see, and the step stays readable and executable.

**At scale:** adopt one canvas shape across every product and the payoff compounds. A product owner who learns to read the R block on one project can sign off on any project. A security reviewer greps every repo for the `S - Safeguards` comment block and audits the whole portfolio in an afternoon. And acceptance criteria stop being rewritten by QA from scratch -- the PO's words land in the R block, the scenarios underneath are visibly derived from them, and that derivation is what makes a green run something a non-engineer can trust as sign-off.

---

## Recipe 337: The SPDD Loop with AI-Generated Code

### Problem
You have the canvas. Now the temptation is to paste fragments of it into a chat window, copy code back out, and eyeball the result -- which is exactly the lossy, unrepeatable loop SPDD exists to kill. The handoff to AI needs to be as structured as the prompt itself: one committed artifact in, one verified suite out, nothing important living only in a chat transcript.

### Solution

The full loop:

```bash
# 1. Author the structured prompt
git add tests/features/password-reset.feature
git commit -m "spec: password reset (REASONS canvas)"

# 2. Hand it to AI
claude "Read tests/features/password-reset.feature.
        Implement the controllers, model, mailer, and views to make every
        scenario pass. Stay inside the Norms and Safeguards. When you finish,
        run the suite."

# 3. AI generates code, runs the suite
# 4. On failure, AI iterates
# 5. On success, you commit code AND any prompt refinements
```

The webship-js suite is the verification step. If the suite is green, the AI has produced code that satisfies the structured prompt. If red, the AI iterates -- guided by the failing step, not by you re-explaining the spec.

Notice what step 5 says: commit the code *and any prompt refinements*. Generation almost always teaches you something about the spec -- an ambiguous Norm, an entity field nobody mentioned. Folding that back into the feature file is the difference between SPDD and a one-shot generation with extra ceremony.

### Where webship-js shines
The Cucumber output names the failing scenario, the failing step, and the expected/actual -- and the default profile also writes a machine-readable JSON report to `tests/reports/cucumber_report.json`. An AI agent does not need to parse console colors; it reads the JSON, finds the failed step, and patches the code. That same JSON is what `bin/generate-reports.js` turns into the HTML and PDF a human reads, so the artifact the AI iterates against and the artifact a delivery manager signs off on are the same file -- one source, two readers. The default 45-second step timeout is a quiet safety net here too: generated code that hangs gets killed and reported as a failure instead of stalling the loop. No human bottleneck.

> **Pro tip:** Tag the most important scenarios `@critical`. Run `npx cucumber-js --tags "@critical"` first in CI. Fast feedback for the SPDD loop.

**At scale:** the loop is cheap to run once and expensive to run carelessly across a portfolio. Every generation pass is tokens, so the teams that survive a budget review run the *deterministic* half everywhere and the *generative* half selectively: the `@critical` cucumber-js run gates every PR in every repo, while the "hand it to AI and regenerate" pass is reserved for the long-lived products where regeneration is actually practiced. Wire the same `--tags "@critical"` profile into a shared CI template so "fast feedback" means the same scope on all twenty suites, and tag the JSON report with `--app-version` and `--env` metadata so a green run on product C is comparable to a green run on product K -- the difference between a portfolio quality signal and twenty incomparable thumbs-up.

---

## Recipe 338: When Reality Diverges, Fix the Prompt First

### Problem
Tuesday morning, a bug report: password reset emails sometimes land in spam. The engineer on triage can patch the mailer config in ten minutes, and every instinct says do exactly that. But the ten-minute fix updates the code and leaves the prompt ignorant -- the spec now describes a system that no longer exists, and the next regeneration will faithfully rebuild the bug. Where does the change start?

### Solution

**Not in the code.** SPDD's most important rule:

> **Important:** When reality diverges from the prompt, fix the prompt first. Then regenerate or update the code.

Steps:

1. Open `tests/features/password-reset.feature`.
2. Update **Norms** to add: "All transactional email must include valid SPF, DKIM, and DMARC headers."
3. Add a new scenario verifying the headers:

```gherkin
@critical @auth @deliverability
Scenario: Reset email passes SPF / DKIM / DMARC
  When I fill in "Email" with "user@example.com"
  And I press "Send reset link"
  And I wait for AJAX to finish
  Then the API response status code should be 200
  And the most recent email to "user@example.com" should pass SPF
  And the most recent email to "user@example.com" should pass DKIM
  And the most recent email to "user@example.com" should pass DMARC
```

4. Run the suite. The new scenario fails (no implementation yet).
5. Hand the updated prompt to AI, or implement manually.
6. Suite goes green. Commit the prompt change AND the code change in one commit.

### Why this matters
A code-first fix would patch the mailer config and forget to update the spec. Six months later, a refactor removes the SPF setup, the spec says nothing about it, and the bug returns. Prompt-first fixes prevent that decay.

Watch the order of operations in step 4: the new scenario must fail *before* anyone implements anything. That red run is evidence the check actually checks something -- the same red-first discipline TDD has always used, applied to a prompt. The email-header steps themselves (`should pass SPF` and friends) are custom step definitions your team writes once against a mail-capture service; after that, every deliverability Norm in every product is one Gherkin line away from being enforced.

**At scale:** prompt-first fixes are greppable, and that is their quiet superpower. When the same spam-folder bug shows up on a second product, you search every repo's feature files for "DKIM" and get an instant map of which products enforce deliverability and which never got the Norm. A delivery manager asking "did the fix from product A reach products B through F?" gets an answer from grep, not from interviewing six teams.

---

## Recipe 339: Three SPDD Skills Mapped to Webship-js

<!-- diagram: spdd-skills -->

Thoughtworks names three skills that make SPDD work. Each maps directly to a webship-js practice:

### 1. Alignment -- "Lock intent before you write code"

In webship-js: write the **Feature: ... As ... I want ... So that** narrative *before* any scenarios. This is the alignment moment. Show it to product, design, security. Get the nods. Only then write scenarios. The narrative costs five minutes to change; the forty scenarios underneath it cost a sprint. Arguing at the cheap layer is the whole point -- and it is the last moment a product owner can redirect the work by editing three lines of plain English.

### 2. Abstraction-First -- "Design before you generate"

In webship-js: define the **Background** and the **selectors** before writing the per-scenario steps. Decide what entities you are talking about. Decide what page objects or named selectors will exist. Write a "Given I define css selectors:" data table. Now scenarios slot into a clean abstraction. This matters double when AI generates the scenarios: a model handed an existing selector vocabulary reuses it, while a model handed nothing invents a fresh CSS selector per scenario -- and you inherit the cleanup.

### 3. Iterative Review -- "Output as a controlled loop"

In webship-js: never let AI run unattended for more than a single feature at a time. After every generation, review:

- Did the new scenarios respect the existing Background?
- Did the AI introduce new selectors when existing ones would have worked?
- Did any Safeguard get bypassed?

Then iterate. Tighten the prompt. Regenerate. Repeat. The review questions above are checkable in a diff -- "did any Safeguard get bypassed?" is a question about scenarios and tags, not about reading every generated line of implementation. That is what keeps the loop fast enough to actually run.

**At scale:** these three skills are worth standardizing as review gates, not leaving to each team's taste. Write the three review questions into the PR template every repo shares, and "did the AI invent a new selector?" stops depending on whether the reviewer happened to think of it. The selector half of that question gets sharper when you back it with a committed `tests/selectors/` registry per product: a model handed an existing selector vocabulary reuses it, so the bigger the registry the team curates, the less cleanup any single generation produces. When all twenty products run the same alignment-then-abstraction-then-review loop, a new QA lead moving between them inherits one workflow instead of twenty dialects -- and a manager auditing "are teams reviewing AI output, or rubber-stamping it?" reads PR histories with the same three questions in every repo.

---

## Recipe 340: SPDD-Style Prompt Templates for AI Agents

### Problem
The canvas worked -- and then it spread. Three sprints in, five engineers and two AI agents are authoring feature files, and every one of them remembers REASONS slightly differently. One drops Entities. One merges Norms into Safeguards. The agent, asked from a blank context, invents an eighth section. Reviews slow to a crawl because every file has its own shape, and the new hire's first PR copies whichever broken variant they saw first. The structure needs to come from the repo, not from memory.

### Solution

Drop a `templates/spdd-feature.md` in your repo:

```markdown
# Feature name

## R - Requirements
- As a [role]
- I want [capability]
- So that [outcome]
- Definition of done: [bullet list]

## E - Entities
- [Noun]: [fields]

## A - Approach
- [One paragraph]

## S - Structure
- Pages / Routes:
- API:
- Components:

## O - Operations
- [List of scenarios to write]

## N - Norms
- i18n:
- Accessibility:
- Performance:
- Logging:

## S - Safeguards
- Security:
- Privacy:
- Rate limits:
- Failure modes:
```

Add to `.claude/CLAUDE.md`:

```markdown
When asked to author a webship-js feature file:
1. Read templates/spdd-feature.md.
2. Fill every section before writing any Gherkin.
3. Place the filled-in REASONS canvas as comments at the top of the .feature file.
4. Generate one scenario per item in the Operations section.
5. Tag every scenario with the relevant Norm/Safeguard category.
```

Now every AI-driven feature file starts from the same canvas. The split matters: the template is *data* (what a finished canvas looks like) and the CLAUDE.md instruction is *policy* (when and how to use it). An agent given both cannot improvise the structure away. Rule 2 -- fill every section before writing any Gherkin -- forces the same order the canvas teaches humans: alignment first, operations last. An agent that cannot fill in Entities has found a gap in the story, not a formatting problem, and it surfaces that gap before a single scenario exists. Rule 5 wires Norms and Safeguards into tags, which means the governance sections stop being comments and start being things `--tags` expressions can run.

Because the template itself is committed, it is versioned prompt infrastructure. When the security team decides every Safeguards section needs a "data retention" line, that is one PR to one file -- and every feature authored after it inherits the change. Compare that to chasing five engineers' personal prompt stashes.

**At scale:** the template stops being per-repo furniture and becomes a shared dependency -- a small package, or a file your project scaffolder drops into every new repo -- so the canvas shape is identical on product 1 and product 21 without anyone copying it by hand. That single point of change is the whole payoff. When governance evolves, the security team opens one PR against the shared template, every project that pulls the update authors its next feature with the new Safeguards line, and a compliance auditor checks one diff instead of interviewing twenty teams about their prompt conventions. The cost the pattern removes is the silent one: twenty slightly different canvases, none of them wrong, none of them comparable, each one a small tax on every review and every onboarding.

> **Pro tip:** Pair this with the `webship-js-ai-agent` Claude subagent. The agent already knows the step definitions; the template gives it the structure.

---

## Recipe 341: The webship-js Stage Map

A team that has just bought into SPDD always asks the same question in the first week: "fine, but which webship-js thing implements which stage?" The new QA lead, onboarding onto their third product, does not want philosophy -- they want to know where intent lives, where scope lives, and what command proves the prompt is satisfied. Without an agreed map, each project answers differently, and the discipline dissolves into local dialects.

The webship-js step library makes Operations easy to express. Here is the stage map most teams converge on:

| SPDD stage          | Webship-js mechanism                                           |
|----------------------|----------------------------------------------------------------|
| Capture intent       | `Feature:` description + `As/I want/So that`                   |
| Lay down domain      | Comment block before `Background`                              |
| Set scope            | `@tags` on scenarios (`@critical`, `@auth`, `@a11y`)           |
| Generate operations  | `Scenario:` blocks with Given/When/Then steps                  |
| Encode norms         | `worldParameters` config + locale/a11y scenarios               |
| Encode safeguards    | `@security` tags + assertion scenarios + AJAX-wait discipline  |
| Verify              | `npm run test:chromium`                                        |
| Refine prompt       | Edit feature file, commit, regenerate code                     |

Two rows do quiet heavy lifting. The verify row is deliberately a single command: `npm run test:chromium` pins the browser through the `BROWSER` env var, so "the prompt is satisfied" means the same thing on a laptop and in CI -- no argument about which run counts. And the safeguards row lists "AJAX-wait discipline" alongside tags because webship-js enforces it mechanically: after every state-changing step, an AfterStep hook runs smartSettle, which watches the instrumented fetch/XHR counter, pending timers, and DOM mutations until the page is genuinely quiet. A safeguard scenario that asserts on a half-rendered page would be a safeguard in name only; the settle engine is what makes the assertion mean something.

This is SPDD with webship-js bolted on. The discipline scales to teams.

**At scale:** the map is the thing you standardize across a portfolio, not the individual feature files. When all twenty projects use the same stage map, a delivery manager can ask "show me the Safeguards for product X" and get the same artifact shape every time -- tags plus assertion scenarios, greppable, runnable. Cross-project questions ("which products encode an accessibility Norm?") become one search over `@a11y` instead of twenty status meetings. Print the table, pin it in the team wiki, and treat deviations from it as review findings.

---

## When to Use SPDD with Webship-js

> **Best practice:** SPDD pays off when stakes are high and lifetime is long. Skip it for prototypes.

**High-value scenarios (use SPDD):**

- Regulated systems (finance, healthcare, government).
- Enterprise CMS where business logic accumulates over years.
- Customer-facing flows with security or accessibility requirements.
- Any feature whose behavior must remain stable across releases.

**Low-value scenarios (skip SPDD):**

- Throwaway prototypes.
- One-off internal scripts.
- UI explorations where you have not yet decided what the feature does.

The cost of SPDD is the time to fill the canvas. The benefit is decades of preserved intent. Pick the projects where the benefit clears the cost.

---

## SPDD vs Spec-Driven Development

You will see related terms. The differences matter:

| Practice                          | What it asks                                              |
|-----------------------------------|-----------------------------------------------------------|
| **Vibe coding**                   | Chat with AI until something works.                       |
| **Spec-driven development (SDD)** | Write specs as Markdown, hand to AI, generate code.       |
| **Structured prompt-driven (SPDD)** | Write specs in a fixed canvas (REASONS), version them, regenerate when prompts change. Code is regenerated; the prompt is the source of truth. |
| **Test-Drive-Develop (this book)**| Write executable specs (Gherkin), AI generates code, tests verify. Prompt + tests are joint sources of truth. |

Webship-js sits at the intersection of SPDD and Test-Drive-Develop. The Gherkin file *is* the structured prompt. The step definitions *are* the executable verification. The pair is unbreakable -- if the test passes, the code matches the spec by definition.

---

## Recipe 342: SPDD-Driven CI Gate

### Problem
The discipline holds for exactly as long as every engineer holds it. Then comes the Friday-afternoon hotfix: code patched, suite green, prompt untouched -- and nobody notices, because nothing *checks*. Six weeks later someone regenerates from the feature file and faithfully rebuilds the pre-hotfix behavior. Fix-the-prompt-first cannot live on goodwill alone; it needs a gate that fails the merge when prompt and code have quietly parted ways.

### Solution

Add a CI step that:

1. Re-runs the AI generation in dry-run mode against the current `.feature` files.
2. Diffs the output against the committed code.
3. Fails the job if the diff is non-empty.

```yaml
# .github/workflows/spdd-check.yml
name: SPDD prompt-code sync

on: [pull_request]

jobs:
  spdd-sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run SPDD generator (dry-run)
        run: |
          npx claude-code --headless \
            --prompt "Read every tests/features/*.feature file. Confirm the implementation in src/ matches every scenario. List any divergences."

      - name: Run webship-js suite
        run: npx cucumber-js --tags "@critical"

      - name: Comment PR with SPDD report
        if: failure()
        run: gh pr comment ${{ github.event.pull_request.number }} --body "SPDD divergence detected. Fix the prompt first, then regenerate code."
```

The gate has two halves, and they fail differently. The cucumber-js run is deterministic: `--tags "@critical"` executes only the scenarios you marked as merge-blocking, each step gets the default 45-second timeout, and a failure means the code demonstrably does not do what the prompt says. The AI divergence check is probabilistic: it asks the model to *read* rather than write -- compare every scenario against `src/` and list mismatches -- which catches drift the suite cannot, like a Norm ("rate-limited 3/hour") that no scenario asserts yet. Trust them accordingly. The deterministic half can block merges from day one; the probabilistic half should run nightly first, until you have seen its false-positive rate with your own eyes. The PR comment matters more than it looks: it tells the engineer *what to do* -- fix the prompt first -- at the exact moment they are about to do the opposite.

This is heavier than a normal CI gate. Run it nightly in early adoption, on every PR once the team trusts it.

**At scale:** the LLM half of this gate has a per-run token cost, and across forty repositories on every PR that becomes a real line item. The pattern that survives the finance review: deterministic `@critical` gate on every PR in every repo (cheap, fast, zero false positives), prompt-sync check nightly and only on the products where regeneration is actually practiced -- usually the regulated, long-lived ones from the "When to Use SPDD" list. Publish the workflow as a shared template so the gate itself cannot drift project by project, which would be a faintly ridiculous way for an anti-drift gate to fail.

---

## The SPDD Mindset

The most surprising thing about SPDD is how little it changes about the code. You still write code. You still test. You still ship. What changes is **what you treat as primary**.

Before SPDD: code is primary, specs are documentation.
After SPDD: prompts are primary, code is the artifact the prompt produces.

This is not academic. It changes how you debug. It changes how you onboard. It changes how AI-driven refactors stay safe over years.

Webship-js was built for the same world. Every step definition is a verb. Every feature file is a contract. Stack SPDD on top, and you have a system where:

- Prompts are committed.
- Tests are executable.
- AI generates code.
- Tests verify.
- Failures point at the prompt, not at the engineer's coffee intake.

That is the discipline. That is the loop. Master it.

---

## Chapter Summary

- SPDD treats prompts as first-class delivery artifacts: version-controlled, reviewed, reused.
- The **REASONS canvas** structures every prompt: Requirements, Entities, Approach, Structure, Operations, Norms, Safeguards.
- Webship-js feature files are structured prompts in disguise -- the REASONS canvas maps cleanly to feature comments + Gherkin.
- Three required skills: **Alignment** (lock intent first), **Abstraction-First** (design before generation), **Iterative Review** (controlled loops, not one-shot prompts).
- The cardinal rule: **when reality diverges, fix the prompt first; then update the code.**
- High-value SPDD scenarios: regulated, long-lived, complex business logic. Skip it for prototypes.
- Webship-js sits naturally at the SPDD + Test-Drive-Develop intersection -- Gherkin is the prompt, step definitions are the verification.
- SPDD-style CI gates can re-verify prompt-code sync on every PR.

---

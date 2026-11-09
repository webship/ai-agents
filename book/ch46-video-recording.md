# Chapter 46: Video Recording Deep-Dive

---

A failing test that prints a stack trace is useful. A failing test that prints a stack trace **and** a 30-second video of what the browser was doing is invaluable. A flake that happens only on the CI runner -- not on your laptop, not on the dev VM -- becomes solvable the moment you can watch it. Webship-js 2.0 added a native `recordVideo` integration: a `WEBSHIP_VIDEO` environment variable with four modes (`off`, `on`, `on-failure`, `tag`), `@video` and `@no-video` tags for per-scenario overrides, and start/stop/save steps wired to Playwright's `BrowserContext.recordVideo` options.

This chapter is the full deep-dive: configuration, retention policy, sizes, hook patterns, CI-artefact wiring, and the trade-offs between recording everything and recording only failures.

---

## How Playwright Records Video

<!-- diagram: page-lifecycle -->

Playwright's video recorder runs at the browser context level, not the page level. When `recordVideo.dir` is set on the context options, every page in that context streams to a `.webm` file. The recording starts when the page is created and stops when the context closes -- then, and only then, does Playwright flush the file to disk. There is no "pause", no "resume", and no retroactive capture. Recording is a property of the context, decided at the moment the context is built. That one fact explains almost everything else in this chapter, including the warning that closes this section.

Webship-js wraps this with an environment-variable control surface and a handful of Gherkin-friendly steps, so you rarely touch the raw Playwright options. The env vars first:

- `WEBSHIP_VIDEO` -- `off` (the default), `on` (record every scenario), `on-failure` (record everything, then keep only the runs that failed), or `tag` (record only `@video`-tagged scenarios). One variable, four shapes, set per environment instead of edited per project.
- `WEBSHIP_VIDEO_DIR` -- where files land; defaults to `./videos`. The directory is created if it does not exist.
- Recording size defaults to 1280x720. Files are named by the pattern `{datetime}.{feature_file}.{scenario}.{status}.{ext}` -- timestamp, source feature, scenario name, and pass/fail status, all readable from a directory listing without opening a single file. That status token in the filename is the difference between scanning a folder and watching twenty videos to find the one failure.

Two tags override the global mode per scenario, no code required: `@video` forces recording on even when the mode is `off`, and `@no-video` suppresses it even when the mode is `on`. And the steps you will use most:

- `Given I start video recording` -- creates a recording-enabled context if one isn't already active
- `When I stop video recording` -- finalises the current recording and flushes the `.webm` to disk
- `When I save the current video as "name"` -- moves the finished recording to a chosen filename

A fresh context is created per scenario by default, so videos are naturally per-scenario -- one scenario, one file, no bleed between tests.

Now the warning, because it bites everyone exactly once and then they remember it forever. Since recording is fixed at context creation, the mid-scenario `start`/`stop` steps cannot flip a switch on a live context. They tear the current context down and build a new one. **Page state is lost the moment you call them**: your logged-in session, the cookies in `context.cookies()`, localStorage, the current URL, the DOM you just settled. If you `start video recording` after logging in, the new context is logged out and sitting on a blank page. The rule that follows from this is simple -- make `Given I start video recording` the *first* step of the scenario, before any navigation or authentication, or set `WEBSHIP_VIDEO=on` and let the per-scenario context record from birth so you never call the mid-scenario steps at all. Everything that happened before the start step is gone, and no error will tell you so; you will just see a video that begins from a clean slate.

---

## Recipe 511: Recording an Entire Scenario

### Problem
Thursday's sprint review needs a walkthrough of the booking flow, and you are about to screen-record yourself clicking through it by hand -- for the fourth sprint in a row, because the flow keeps changing and last sprint's recording is already wrong. Meanwhile the suite walks that exact flow every night. The recording should be a by-product of the test, not a chore that competes with it.

### Solution

```gherkin
Feature: Documentation reel

  Scenario: Booking happy path
    Given I start video recording
    And I am on "/"
    When I click "Book now"
    And I fill in "name" with "Sara"
    And I fill in "dates" with "2026-06-01 to 2026-06-05"
    And I press "Search"
    And I click "Reserve" in the "Sunset Suite" row
    And I press "Confirm"
    Then ".confirmation-number" should be visible within 10 seconds
    And I stop video recording
```

### How It Works

`Given I start video recording` is the first step on purpose. It builds a recording-enabled context, and because it is first, there is no page state to lose -- nothing has logged in, nothing has navigated yet. From there the scenario runs as any other: `And I am on "/"` settles the page, the form fills resolve labels first, and `Then ".confirmation-number" should be visible within 10 seconds` polls every 100ms up to the deadline rather than sleeping. `And I stop video recording` finalises the recording and flushes the `.webm` to disk. The file lands in `./videos` named by the `{datetime}.{feature_file}.{scenario}.{status}.{ext}` pattern, so a passing run produces something like `2026-06-12T14-30-00.documentation-reel.Booking-happy-path.passed.webm` -- you can read what it is, when it ran, and that it passed, all from the listing.

Here is the thing worth internalising: the recording follows the same waits the test follows. Because webship-js never inserts static sleeps -- every state-changing step ends with `smartSettle`, which waits for `window.__webshipAjaxCount` to hit zero and the DOM to go quiet for 250ms -- the video shows the app at rest at each step, not a stuttering blur of half-loaded pages. The reel is watchable precisely because the test is honest about waiting.

**At scale:** The reason this beats hand-recording is not that it saves five minutes once. It is that the recording can never drift from reality. When you run the same documentation scenario across twenty client sites, each one re-records itself every night against the live build -- the demo reel for project A and project B are always current, always reflecting the deployed flow, with zero manual upkeep. A PO preparing a sprint review pulls the latest `.webm` from the artefact bundle instead of asking QA to re-screen-record. Standardise on `WEBSHIP_VIDEO=on` for a dedicated "demo" tag-set and the whole portfolio gets self-updating walkthroughs for free.

**Pro Tip:** Keep documentation scenarios in their own feature file (or behind a `@demo` tag) and run them separately from your regression suite. They are slower by design -- you may even slow them down deliberately (Recipe 522) -- and you do not want that wall-time tax on the suite that gates your releases.

---

## Recipe 512: Recording With a Custom Name

### Problem
A customer reports a checkout bug, you reproduce it, and you want to hand the engineer a video they can watch -- not a wall of text. The default `{datetime}...{scenario}.{status}` filename is great for a directory you scan, but when you are attaching one file to one Jira ticket, you want the ticket ID right there in the name so it never gets lost in a `videos/` folder full of timestamped runs.

### Solution

```gherkin
  Scenario: Repro for ticket BUG-1284
    Given I start video recording
    And I am on "/checkout"
    When I click "Apply" without entering a coupon
    And I save the current video as "BUG-1284-empty-coupon.webm"
```

### How It Works

`When I save the current video as "name"` waits for the active recording to finalise, then moves the finished `.webm` to the filename you chose. It overrides the automatic `{datetime}.{feature_file}.{scenario}.{status}` pattern for this one file only -- every other recording in the directory keeps its default name and is left untouched. Drop the result into the bug-tracker ticket and the engineer has a perfect repro to work from.

Mind the ordering, and remember the warning from the chapter opener: `Given I start video recording` is the first step here because it rebuilds the context. If you had logged in or navigated before it, that state would be discarded and the recording would start from a blank page. Start first, then navigate. The save step is the natural last step -- once you call it, the recording is closed and finalised.

**Pro Tip:** Parameterise the name. If your runner injects the ticket ID as an env var or a scenario tag, an After hook can build `${ticketId}-${status}.webm` automatically (the mechanics are in Recipe 518). Then anyone reproducing a bug just tags the scenario and the evidence files itself under the right ID.

---

## Recipe 513: Recording Only on Failure

### Problem
Your nightly run has 500 scenarios. Recording every one produces 5GB of `.webm` per run, most of it footage of tests passing -- which nobody ever watches. But the one scenario that fails intermittently on the CI runner, the one you cannot reproduce on your laptop, is exactly the footage you would give a week of your life for. You want video for failures and nothing else.

### Solution
The simplest path is the built-in mode -- set `WEBSHIP_VIDEO=on-failure` and webship-js records every scenario but keeps only the ones that fail, deleting the passing recordings for you. When you want control over the kept filename, do it explicitly with hooks:

```js
const { Before, After } = require('@cucumber/cucumber');
const fs = require('fs');

Before(async function(scenario) {
  this.videoEnabled = true;
  await this.startVideoRecording();
});

After(async function(scenario) {
  const videoPath = await this.stopVideoRecording();
  if (scenario.result.status === 'PASSED') {
    fs.unlinkSync(videoPath);   // discard on success
  } else {
    const safe = scenario.pickle.name.replace(/[^a-z0-9]/gi, '_');
    fs.renameSync(videoPath, `videos/FAIL_${safe}.webm`);
  }
});
```

### How It Works

`WEBSHIP_VIDEO=on-failure` is the mode to reach for first -- it records into the per-scenario context from birth (so no page state is ever lost) and, in its After hook, inspects `scenario.result.status` and unlinks the file on `PASSED`. The hook version above does the same dance manually, which is what you want when you need a custom name like `FAIL_<scenario>.webm` rather than the default pattern. Note the trade-off the env mode hides: recording everything and discarding the passes still pays the 5-10% wall-time cost on *every* scenario, because Playwright must record before it knows the outcome. You cannot record only failures without recording everything first -- there is no retroactive capture, as the chapter opener warned. The saving is in storage and signal, not in runtime.

**At scale:** This is the single most valuable video setting for a team running many suites, so make it the default and stop thinking about it. Bake `WEBSHIP_VIDEO=on-failure` into the CI environment for the whole portfolio -- one variable in the platform config, not a code change in forty repos. Every project's nightly run then yields a `videos/` folder that contains only failures, which collapses triage from "watch everything" to "watch the three that broke." Wire that folder into the artefact upload with `if: always()` and a QA lead can open last night's failures for any project without re-running a thing. The cross-project win is uniformity: when every suite keeps failure videos under the same convention, your PMO dashboard can link straight to the evidence for a red build, and "is release X safe?" stops being a question of trust and becomes a folder someone can open.

**Pro Tip:** Pair this with the screenshot-on-failure default (`WEBSHIP_SCREENSHOT_ON_FAILED` is `true` out of the box). A failure then leaves both a final-state PNG and a video of how it got there -- the snapshot answers "what broke" in one glance, the video answers "why."

---

## Recipe 514: Recording Only With a Tag

### Problem
Most of your scenarios do not need video -- they pass, nobody watches them, and the 5-10% wall-time cost adds up across a suite. But a handful are worth recording every single run: the flaky one you are still chasing, the payment flow legal wants on file, the smoke test a PO checks before sign-off. You want those recorded on every run and the rest left alone, decided in the feature file by whoever writes the scenario -- not buried in a config someone forgets exists.

### Solution
The built-in way needs no code at all -- set `WEBSHIP_VIDEO=tag` and tag the scenarios you care about:

```gherkin
  @video
  Scenario: Reproduce ticket 1284
    Given I am on "/checkout"
    ...
```

If you want a different tag name or extra logic, write your own hook pair:

```js
Before({ tags: '@record' }, async function() {
  await this.startVideoRecording();
});

After({ tags: '@record' }, async function() {
  await this.stopVideoRecording();
});
```

```gherkin
  @record
  Scenario: Reproduce ticket 1284
    Given I am on "/checkout"
    ...
```

### How It Works

`WEBSHIP_VIDEO=tag` tells webship-js to build a recording-enabled context only for scenarios carrying `@video`, and to leave everything else on a plain context -- so untagged scenarios skip the overhead entirely. Because the decision is made when the context is created, the tagged scenario records from its first step with no page state lost; you do not call the mid-scenario start step, so the warning from the opener does not apply here. The custom `@record` hook above is the same idea wired by hand: Cucumber runs the `Before` only for scenarios matching the tag expression, so `startVideoRecording` fires before any of your steps. Two notes on the built-in tags worth keeping straight -- `@video` forces recording *on* even when the mode is `off`, and `@no-video` forces it *off* even when the mode is `on`. So `WEBSHIP_VIDEO=on` plus `@no-video` on three sensitive scenarios is the inverse pattern: record everything except the ones you opt out of.

**At scale:** Tag conventions are how a multi-project team keeps video sane without per-project config. Agree on one vocabulary -- `@video` for "always record this", `@no-video` for "never record this, it is sensitive" -- and apply it identically across every suite. New hires learn it once and it reads the same in every repo. A product owner who cannot edit a hook file *can* add `@video` to the acceptance scenario they care about, which means the people writing the criteria control the evidence without filing a ticket to engineering. Run with `WEBSHIP_VIDEO=tag` as the portfolio default and your storage bill tracks the handful of tagged scenarios, not the thousands that pass quietly every night.

**Pro Tip:** Reserve `@no-video` for scenarios that touch PII or payment data even when your default mode is `off`. A future engineer who flips the suite to `WEBSHIP_VIDEO=on` for a debugging session will then automatically skip recording the sensitive flows -- the safety is in the tag, not in remembering the rule.

---

## Recipe 515: Custom Video Size

### Problem
Your CI artefact bucket has a quota, and the platform team just sent the "your storage is at 90%" email. The 1280x720 default is crisp, but you are keeping failure videos across forty nightly suites and the maths does not work. Drop to 800x600 and you roughly halve the bytes per recording -- still legible enough to watch a flake, small enough that the bucket stops filling.

### Solution
Set the context options in your World:

```js
// PlaywrightWorld.js
this.context = await browser.newContext({
  recordVideo: {
    dir: 'videos/',
    size: { width: 800, height: 600 },
  },
});
```

### How It Works

`recordVideo.size` is independent of the browser viewport. Playwright records the page at the actual viewport, then scales each frame down to the size you asked for before encoding -- so the layout is unchanged, only the resolution of the saved `.webm` drops. The file-size table later in this chapter puts numbers on it: 1280x720 runs 1-3 MB per minute, 800x600 runs 0.4-1 MB, and 640x480 runs 0.2-0.5 MB. Halving each dimension does not exactly halve the file -- video compression is non-linear -- but the saving is real and consistent. Below 640x480 the trade tips the wrong way: error toasts and form labels turn to mush and the recording stops being useful for the one job it has.

**At scale:** Resolution is a fleet-wide knob, so set it once and centrally. Rather than editing `recordVideo.size` in forty `PlaywrightWorld.js` files, prefer the env-var surface (`WEBSHIP_VIDEO_DIR` and the `WEBSHIP_VIDEO` mode) and treat size as a deliberate, documented override in projects that need it. A practical pattern for a portfolio: record CI failures at 800x600 to keep the nightly storage bill flat, and reserve 1280x720 for the `@demo` scenarios a PO or designer will actually present. Multiply 0.5 MB saved per minute across 200 failure videos a night across many projects and the platform team's quota email stops arriving.

**Pro Tip:** Decide resolution by audience, not by habit. Forensic failure footage only needs to be legible to an engineer scrubbing for the broken step -- 800x600 is plenty. Footage going in front of a client or a sprint review earns the full 1280x720. Tag-gate the two cases so each scenario gets the size its viewers need.

---

## Recipe 516: Recording at the Viewport Size

### Problem
A bug only shows up on mobile -- the menu overlaps the logo at the small breakpoint -- but your videos all record at a fixed 1280x720, so the recording shows a desktop layout that looks fine. The footage and the bug disagree, and the engineer watching it cannot see what the user saw. You want the video to match the exact viewport the test ran at, so a mobile test produces mobile-shaped footage.

### Solution
Omit `size` and Playwright records at the page's viewport size:

```js
this.context = await browser.newContext({
  viewport: { width: 1366, height: 768 },
  recordVideo: { dir: 'videos/' },   // size omitted -> uses viewport
});
```

### How It Works

When `recordVideo.size` is absent, Playwright falls back to the context's viewport for the recording dimensions -- so the `.webm` is shaped exactly like the page under test. Pair this with webship-js's breakpoint registry and you get per-breakpoint recordings without thinking about it. The registry ships seven named widths -- `xs` 375x667, `sm` 576x800, `md` 768x1024, `lg` 992x768, `xl` 1200x900 (the default), `xxl` 1400x900, and `xxxl` 1920x1080 -- so a scenario that runs at `xs` records a 375-wide vertical video that looks like a phone, while the same scenario at `xl` records a wide desktop frame. The footage matches the failure because it matches the viewport that produced it. (Override the registry per environment with the `WEBSHIP_SELECTORS_BREAKPOINTS` JSON if your design system uses different widths.)

**At scale:** Responsive coverage is where this earns its keep across a portfolio. If your suites run the same flows at multiple breakpoints -- and for a multi-site team they should -- viewport-sized recordings give you a visual record at every width, so a mobile-only regression on project C is obvious in its own footage instead of hidden behind a desktop frame. Standardise the breakpoint names across projects (the same `xs`/`sm`/`md` vocabulary everywhere) and a reviewer can pull "the mobile video for the checkout failure" on any site and know exactly what they are getting. One caveat for the storage-conscious: viewport recordings vary in size, and a 1920x1080 `xxxl` run is heavier than a fixed 800x600 -- so reserve full-viewport recording for the breakpoints where layout actually matters rather than every width on every project.

---

## Recipe 517: Saving Videos to a Custom Directory

### Problem
Your CI job uploads `tests/reports/` as the build artefact, and a reviewer downloads it expecting everything they need. But the videos land in a top-level `videos/` folder that the upload step never touches -- so the report is there, the failure videos are not, and someone has to go re-run the job locally to watch the flake. You want the videos to live inside the evidence tree that already ships, so one archive holds the report, the screenshots, and the recordings together.

### Solution
Point the directory at your evidence tree -- via the env var when you can, in the World options when you cannot:

```bash
WEBSHIP_VIDEO_DIR=tests/evidence/videos npm test
```

```js
this.context = await browser.newContext({
  recordVideo: { dir: 'tests/evidence/videos/' },
});
```

Then wire that one tree into the artefact upload alongside `tests/reports/`:

```yaml
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: test-evidence
    path: tests/evidence/
```

### How It Works

`WEBSHIP_VIDEO_DIR` is the supported way to relocate recordings -- it defaults to `./videos`, and webship-js creates the directory if it does not exist, so you do not need a `mkdir` step. Setting it to a path under `tests/evidence/` means recording, report, and screenshots share one root. The `if: always()` on the upload is the load-bearing detail: without it, a failed test step aborts the job before the upload runs, and the failure footage -- the only footage you actually wanted -- never leaves the runner. With it, the archive uploads on pass or fail, so red builds keep their evidence. One download, and the reviewer has the cucumber HTML report, the failure PNGs, and the `.webm` recordings in a single tree.

**At scale:** A single evidence root per project is what makes cross-project triage bearable. When every suite in the portfolio writes report, screenshots, and video under the same `tests/evidence/` layout, the artefact-collection step in your CI templates is identical everywhere -- copy one `upload-artifact` block into forty pipelines and never special-case a project. Set `WEBSHIP_VIDEO_DIR` in the shared CI environment rather than hardcoding the path in each `PlaywrightWorld.js`, and the layout stays uniform even as projects come and go. A PMO that knows "the evidence bundle is always the `test-evidence` artefact, same shape on every job" can point a release-readiness dashboard at it without learning each project's quirks.

**Pro Tip:** Keep the evidence root *outside* your source tree's committed paths and in `.gitignore`. Recordings are build outputs, not source -- they belong in CI artefacts and object storage (Recipe 523), never in the repo where they bloat clones and leak whatever was on screen.

---

## Recipe 518: Per-Scenario Video Path

### Problem
Three months from now, someone opens your video bucket looking for "the smoke-test login failure from mid-May" and finds two thousand files. The default `{datetime}.{feature_file}.{scenario}.{status}` pattern is good, but your team has its own convention -- tag first so smoke and regression sort apart, then status so failures group, then date, then scenario. You want every recording to follow that shape so the bucket is searchable by the names your team actually thinks in.

### Solution
Write an After hook with the full naming logic:

```js
After(async function(scenario) {
  const stat = scenario.result.status.toLowerCase();
  const date = new Date().toISOString().slice(0, 10);
  const name = scenario.pickle.name.replace(/[^a-z0-9]/gi, '-').toLowerCase();
  const tags = scenario.pickle.tags.map(t => t.name.replace(/^@/, '')).join('_');
  const safe = `${tags}_${stat}_${date}_${name}.webm`;
  await this.saveVideoAs(safe);
});
```

### How It Works

Cucumber hands the After hook a rich scenario object, and this recipe mines four parts of it. `scenario.result.status` is the outcome (`PASSED`, `FAILED`, `SKIPPED`) -- lowercased here so failures sort predictably. `scenario.pickle.name` is the human scenario title, slugged to filename-safe characters. `scenario.pickle.tags` is the full tag list, with the leading `@` stripped and joined so `@smoke @auth` becomes `smoke_auth`. Stitched together you get `smoke_auth_failed_2026-05-14_user-login.webm` -- self-describing from the listing alone. `saveVideoAs` then finalises the active recording under that name, the same step Recipe 512 used inline, just driven from a hook so it applies to every scenario without touching the feature files.

This is the manual version of the built-in `{datetime}.{feature_file}.{scenario}.{status}.{ext}` pattern. Reach for the hook only when your team's sort order differs from the default -- if the shipped pattern already sorts the way you think, use it and write no code.

**At scale:** A naming convention is worthless if it lives in forty different hook files that drift apart over time. Put this After hook in a shared step-definition package that every project depends on, so the recording names are byte-for-byte identical across the portfolio. Then a cross-project script -- or a PMO dashboard -- can parse any project's filenames the same way: split on `_`, read tag/status/date/scenario, and build a quality view that spans every suite. Uniform names are what let "show me every failed smoke-test video from last week, all projects" be a `grep`, not an archaeology dig. The moment each team invents its own scheme, that query dies.

**Pro Tip:** Append the CI run ID to the name in CI (`process.env.GITHUB_RUN_ID` or your provider's equivalent). When the same scenario fails on three consecutive nightly runs, run-ID-stamped filenames keep the three recordings distinct instead of overwriting each other -- and you can watch the flake's history rather than just its latest instance.

---

## Recipe 519: Printing the Video Path

### Problem
You are debugging on your laptop, the scenario just ran, and you want to watch the recording right now -- but you have not wired up artefact collection locally and the `{datetime}...` filename is a mouthful to guess. You do not want to `ls -t videos/` and eyeball the newest file. You want the test to just tell you where it put the recording.

### Solution

```gherkin
  Scenario: Locate the video
    Given I start video recording
    And I am on "/"
    When I press "Search"
    Then print video path
```

### How It Works

`Then print video path` is a debug step in the same family as `Then print current URL` and `Then print API response` -- it writes to the test stdout and changes nothing about the run. It resolves the active recording's destination and logs the absolute path, so you can copy it straight into a player (or `Cmd`/`Ctrl`-click it in a terminal that linkifies paths). It is a local-debugging convenience; in CI you would rely on the filename pattern and the artefact upload instead of reading stdout.

One catch tied to the chapter's recurring warning: `Given I start video recording` here rebuilds the context as its first step, so it must come before `And I am on "/"`. Print steps are safe anywhere -- they only read -- but the start step is not, so keep it first.

**Pro Tip:** Leave a `Then print video path` in a scenario you are actively debugging and remove it once the bug is fixed. It is noise in a clean CI log, but during a chase it saves you the `ls` dance every single run. Treat it like a `console.log` -- handy while hunting, gone before you commit.

---

## Recipe 520: Combining Video With Screenshots

### Problem
A scenario fails at "Place Order" and you have the video, so you can watch the whole run -- but to compare the form state before submit against the error state after, you are scrubbing the timeline frame by frame, pausing, squinting. Video is the right tool for "what led here?" and the wrong tool for "what did the page look like at exactly this moment?". You want both: a reel of the journey and crisp stills at the moments that matter.

### Solution

```gherkin
  Scenario: Hybrid evidence trail
    Given I start video recording
    And I am on "/checkout"
    When I fill in "cardNumber" with "4242424242424242"
    And I save screenshot with name "before-submit"
    And I press "Place Order"
    Then ".confirmation-number" should be visible within 10 seconds
    And I save screenshot with name "after-submit"
    And I stop video recording
```

### How It Works

Two evidence systems run side by side and do not interfere. The video recorder streams the entire context at the context level; the screenshot steps capture single frames at the page level, written under `WEBSHIP_SCREENSHOT_DIR` (default `./screenshots`) by their own `{datetime}.{feature_file}.feature_{step_line}.{ext}` pattern. `And I save screenshot with name "before-submit"` drops a still right after the card field is filled and settled, and the matching `after-submit` shot captures the confirmed state. Because every state-changing step ends with `smartSettle`, both screenshots land on a quiet page rather than mid-transition -- the still is sharp, not a blur of a half-rendered modal. The result is a layered trail: stills are the bookmarks, the video is the tape between them.

**At scale:** Two channels of evidence make a failure legible to two audiences at once. An engineer scrubs the video to find the broken step; a PMO or PO who will never open a `.webm` reads the before/after PNGs in the HTML report and understands the failure in one glance. Standardise both across the portfolio -- `WEBSHIP_SCREENSHOT_ON_FAILED=true` (the default) for the automatic final-state still plus `WEBSHIP_VIDEO=on-failure` for the reel -- and every red build in every project arrives with the same dual evidence, no per-scenario authoring required. The named-screenshot steps above are for the handful of flows where you want bookmarks at specific moments, not the default failure capture.

**Pro Tip:** Name screenshots by the state they capture (`before-submit`, `after-submit`), never by step number. Step numbers shift the moment someone edits the scenario; state names stay meaningful through every refactor and read clearly in the report months later.

---

## Recipe 521: Recording in Headless and Headed Modes

### Problem
A new engineer asks the reasonable question: "Do I get video when I run headed on my machine, or only in CI?" Behind it is a real worry -- that the recording is a CI-only feature and local headed runs are a blind spot. You want one answer you can give every new hire: recording works the same in both modes, so the footage is identical whether the browser was visible or not.

### Solution
Playwright records identically in both modes. No special configuration. The same scenario:

```bash
# Headless (CI default)
npm test

# Headed (local) -- HEADLESS=false opens a visible browser
HEADLESS=false npm test
```

Produces the same `.webm` file. The difference: in headed mode, the human can also watch the browser live. The video is the playback for everyone who couldn't.

### How It Works

Video recording is a context property, decided when the context is built -- it has nothing to do with whether the browser draws to a screen. So `HEADLESS=false` (the supported switch; `playwright.config.ts` reads `process.env.HEADLESS !== 'false'`, and `npm run test:headed` sets it for you) changes only whether *you* see the window, not whether Playwright records it. There is one quiet difference baked into `playwright.config.ts` that affects the recording's pacing, though: `SLOW_MO` defaults to 300ms in headed mode versus 0 in CI. A headed run therefore moves a touch slower and its video plays a touch slower, which is usually a feature when you are watching to debug. The frames are the same; only the cadence shifts.

**Pro Tip:** When CI reports a failure you cannot see in the artefact video clearly, re-run that one scenario locally with `HEADLESS=false SLOW_MO=800` -- the canonical webship-js debugging incantation. You then watch it live *and* keep a slow-motion `.webm` of the same run, doubling your odds of catching the moment it breaks.

---

## Recipe 522: Slowing Down for the Camera

### Problem
You record the booking flow for the sprint review, play it back, and it is a blur -- forms fill instantly, the page jumps to confirmation, and a watcher who blinks misses the whole thing. The test is fast because it should be fast, but a documentation reel needs to be *followable*. You want the recording to slow to a human pace without slowing the suite that gates your releases.

### Solution
The supported knob is the `SLOW_MO` env var (`playwright.config.ts` reads it directly), so you can pace a documentation run without touching code:

```bash
SLOW_MO=250 HEADLESS=false npm test    # 250ms between actions
```

If you build the browser yourself in a World, the equivalent launch option is `slowMo`:

```js
this.browser = await chromium.launch({
  headless: false,
  slowMo: 250,   // 250ms delay between actions
});
```

### How It Works

`slowMo` inserts a fixed delay before each Playwright action -- click, fill, press -- so every interaction is spaced out and the recorder captures a watchable cadence instead of instant jumps. It is a pacing layer on top of the waits, not a replacement for them: `smartSettle` still does the real synchronisation (ajax to zero, DOM quiet), and `slowMo` just adds breathing room a human eye can track. Worth knowing about the defaults baked into `playwright.config.ts`: `SLOW_MO` is 0 in CI, 300ms in headed mode, and 800ms when both `HEADLESS=false` and no explicit value are set -- so a plain headed run is already gentler than CI. For documentation, 250-500ms is the sweet spot: fast enough to hold attention, slow enough to follow each click.

For CI, never set `SLOW_MO` above 0 -- it adds wall time to *every* action of *every* scenario, and across a nightly portfolio that compounds into real runner cost for zero testing benefit. Slow motion is a documentation tool, full stop.

**At scale:** Keep the slow, headed, recorded runs in their own lane -- a separate `@demo` profile or a manually triggered pipeline -- and never let `SLOW_MO` leak into the env your regression suites read. The failure mode at scale is a stray `SLOW_MO=300` in a shared CI variable silently taxing forty nightly suites; one engineer added it to debug, forgot to remove it, and now every run is 30% slower. Treat documentation pacing as an opt-in override per run, set on the command line, never persisted into the shared environment.

**Pro Tip:** Pair `SLOW_MO=400` with viewport-sized recording (Recipe 516) and a `@demo` tag for review reels. You get slow, correctly-sized, self-updating footage of every documented flow -- and because it is tag-gated, it never touches the speed of the suite that actually guards your releases.

---

## Recipe 523: Storing Videos to S3 or GCS

### Problem
A flake fails one night in three, and you finally sit down to investigate -- only to find the CI artefact retention is 30 days and the runs you needed expired last week. The footage that would have shown the pattern is gone. For evidence you might revisit months later -- an intermittent failure, a regression a customer reports against an old release -- 30-day artefacts are not enough. You want failure videos in object storage that outlives the CI job that produced them.

### Solution
Add a CI step that uploads after the test step:

```yaml
- name: Upload videos to S3 on failure
  if: failure()
  run: |
    aws s3 sync videos/ s3://my-test-evidence/$(date +%Y-%m-%d)/${{ github.run_id }}/videos/
```

Or for GCS:

```yaml
- name: Upload videos to GCS on failure
  if: failure()
  run: |
    gsutil -m rsync -r videos/ gs://my-test-evidence/$(date +%Y-%m-%d)/${{ github.run_id }}/
```

### How It Works

`if: failure()` is the deliberate choice here, and it is the opposite of the `if: always()` you would use for the artefact bundle. You only push to long-term storage when the run failed -- passing runs leave nothing behind, so the bucket fills with signal, not noise. The path partitions by date *and* `github.run_id`, so every failed run gets its own isolated prefix and nothing ever overwrites earlier evidence -- the run that failed last Tuesday and the run that failed tonight sit in separate folders forever. `aws s3 sync` and `gsutil rsync` both upload only what changed, so the step is cheap even when `videos/` already holds a few files. The bucket becomes a permanent forensic library; link to the run's prefix straight from your bug tracker.

**At scale:** This is where video stops being a debugging toy and becomes an organisational asset. One bucket, partitioned `s3://evidence/<project>/<date>/<run-id>/`, holds failure footage for every project in the portfolio -- so when a customer reports a regression against a release from two months ago, you open the prefix for that project and date and watch what the suite saw, long after the CI artefacts expired. Set a lifecycle policy on the bucket (expire after 90 or 180 days) so storage cost stays bounded without manual cleanup. The DevOps win is that the upload step is identical across pipelines -- only the `<project>` segment changes -- so it drops into your shared CI template once. The PMO win is a single, durable place where "show me the evidence for that failed release" is always answerable.

**Pro Tip:** Print the bucket URL into the CI log right after the upload (`echo "Evidence: https://...$run_id/"`). A reviewer reading the failed job then has a one-click link to the footage instead of reconstructing the path by hand from the date and run ID.

---

## Recipe 524: Cleaning Up Old Videos

### Problem
Your laptop's disk warning pops up mid-sprint, and the culprit is a `videos/` directory you forgot existed -- 20GB of `.webm` from a month of local runs, almost all of it footage of tests that passed and you never watched. Recordings are build output, not history; on your own machine you want them to clean themselves up so they never become the thing that fills your disk.

### Solution
Add a pre-test cleanup:

```bash
# package.json
"scripts": {
  "pretest": "rm -rf videos/*",
  "test": "cucumber-js"
}
```

Or with retention:

```bash
"pretest": "find videos/ -mtime +7 -delete"
```

### How It Works

`pretest` is an npm lifecycle script -- npm runs it automatically right before `test`, so the cleanup happens every run without you remembering to invoke it. The first form wipes `videos/` clean before each run, so you only ever keep the recordings from your latest run; the second keeps anything modified in the last 7 days (`-mtime +7` matches files older than seven days) and deletes the rest, giving you a rolling local window. This mirrors the purge knob the screenshot system exposes (`WEBSHIP_SCREENSHOT_PURGE`) -- same idea, applied to video. Pick the form that matches how you work: wipe-every-run if you only ever care about the run you just did, rolling-window if you sometimes look back a few days.

A word of caution -- this is a *local* convenience, not a CI pattern. In CI you do not purge; you upload (Recipe 517) and archive failures to object storage (Recipe 523). Deleting in CI would throw away the evidence the whole pipeline exists to capture.

**Pro Tip:** Keep CI and local retention deliberately different. CI keeps failure footage for as long as your bucket's lifecycle policy allows; your laptop keeps a 7-day rolling window and no more. Conflating the two -- purging in CI, or hoarding locally -- is how teams either lose evidence they needed or drown in footage they never wanted.

---

## Recipe 525: Auto-Embedding Videos in the HTML Report

### Problem
You hand the HTML report to a product owner so they can sign off on a release, and the report says "Booking happy path -- failed." Their next message is "can you send me the video?" -- and now you are digging through `videos/`, finding the file, attaching it to a chat. Every failure becomes a manual evidence-fetch. You want the recording to play *inside* the report, so opening it shows both the result and the footage in one place, no follow-up needed.

### Solution
Use a custom reporter:

```js
// formatters/video-embedder.js
const { Formatter } = require('@cucumber/cucumber');
const fs = require('fs');

class VideoEmbedder extends Formatter {
  constructor(options) {
    super(options);
    options.eventBroadcaster.on('envelope', (envelope) => {
      if (envelope.testCaseFinished) {
        const videoPath = this.findVideoForCase(envelope);
        if (videoPath && fs.existsSync(videoPath)) {
          this.log(`<video controls width="800"><source src="${videoPath}" type="video/webm"></video>\n`);
        }
      }
    });
  }
  // ...
}
module.exports = VideoEmbedder;
```

Wire it in `cucumber.js`:

```js
module.exports = {
  default: {
    format: [
      'html:tests/reports/cucumber_report.html',
      './formatters/video-embedder.js:tests/reports/videos.html',
    ],
  },
};
```

### How It Works

A Cucumber formatter listens to the run's event stream -- the `envelope` events Cucumber broadcasts as it executes. This one watches for `testCaseFinished`, looks up the recording that belongs to the case that just ended, and, if the `.webm` exists, writes a `<video controls>` tag into its output file. Registering it as a second `format` entry in `cucumber.js` means it runs alongside the standard `html:` reporter without disturbing it -- you get the normal report plus a videos page. This is a custom formatter sitting beside the report toolchain that `bin/generate-reports.js` drives; it does not replace the cucumber-html-reporter output, it augments it.

The closing caveat about lazy-loading is not optional at any real size: a naive embed puts every `<video src>` on one page, and opening it makes the browser fetch all of them at once. For a 200-scenario suite that is a multi-gigabyte page load that hangs the tab. Use `preload="none"` on each `<video>`, or render a poster thumbnail that loads the real file only on click, so the report opens instantly and streams each recording on demand.

**At scale:** The point of an embedded-video report is to put evidence where non-engineers already look. A PO or a PMO opens the HTML report -- the artefact they already trust -- and the failure footage is right there under the failed scenario, no "can you send me the video?" round-trip. Across a portfolio, ship the same formatter in your shared reporting setup so every project's report behaves identically: same layout, same inline playback, same place to click. That uniformity is what lets sign-off scale -- a delivery manager reviewing ten projects' reports before a release learns one report format, not ten, and reads the evidence themselves instead of trusting a green checkmark. Combined with the metadata flags on `bin/generate-reports.js` (`--app-version`, `--env`, `--browser`), each report becomes a self-contained, watchable record of exactly what shipped where.

**Pro Tip:** Embed only failure videos, not every recording. A report with 200 passing-scenario videos is heavy and pointless -- nobody watches a pass. Filter the formatter to `testCaseFinished` events with a failed result, and the report stays light while still carrying the footage that actually matters.

---

## Performance and Sizing

A few rules of thumb from real projects:

| Metric | Typical value |
|--------|---------------|
| Overhead per scenario | 5-10% wall time |
| File size per minute (1280x720) | 1-3 MB |
| File size per minute (800x600) | 0.4-1 MB |
| File size per minute (640x480) | 0.2-0.5 MB |
| Storage per CI run (50 scenarios) | 50-200 MB |
| Storage per month (10 runs/day) | 15-60 GB |

Pick a retention window. Pick a resolution. Tag-gate recordings if you don't need them on every run. The defaults are tuned for "record everything in CI, keep for 30 days" -- adjust if your shape is different.

---

## When to Record and When Not To

Record when:

- You have flakes you can't reproduce locally
- You need design or product review evidence
- You ship to non-engineers who can't read a stack trace
- You're onboarding and want a reference recording

Don't record when:

- The suite already runs fast enough and adds no signal
- Storage is constrained
- The page is sensitive (medical, financial, PII) and the video would create a compliance issue

For the third case, redact: blur fields containing PII before recording, or skip the video step on tagged scenarios.

Video recording is one of the cheapest visibility wins in webship-js -- three Gherkin steps, no infra changes, a tenfold reduction in "I can't reproduce" time. Use it liberally; tune retention conservatively; the team will thank you on the next flake.

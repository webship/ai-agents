# Chapter 40: File Downloads and Video Recording

---

Two features that look unrelated, but both deal with the same thing: artifacts that leave the browser and live on the filesystem. Webship-js ~2.0 added first-class support for asserting that the browser downloaded a file (and what's inside it), plus the ability to record video of the test run for visual evidence.

Downloads are the more practical of the two -- most teams have at least one CSV export, one PDF generator, or one "Save as ZIP" button that needs a test. Video recording is the more dramatic -- when a flake hits in CI, a 30-second video of the test is worth a thousand log lines.

---

## How Downloads Work in Webship-js

<!-- diagram: page-lifecycle -->

A "download" in browser terms is any response that triggers the browser's save dialog -- usually because of a `Content-Disposition: attachment` header or a programmatic anchor with `download="..."`. Playwright surfaces this as a `download` event on the page, and that's what webship-js builds on: the download step arms a listener for that event *before* it triggers the link or URL, so there is no race between "click happened" and "download started". The browser never shows a save dialog (there is no dialog in headless mode anyway); the bytes go straight to a temp directory that Playwright manages, and webship-js keeps a handle to the captured file on the scenario's world object.

You don't need to manage paths. The assertion steps refer to "the downloaded file" -- the most recent download in the scenario. Because every scenario gets a fresh browser context, a download captured in one scenario can never bleed into the next. The suggested filename comes from the server's `Content-Disposition: filename=` header (or the anchor's `download` attribute), which means the name assertions are really *server contract* assertions -- they catch the backend change, not just the frontend one.

---

## Recipe 387: Downloading a File from a Link

### Problem
The "Download CSV" link on your reports page has been there since the first release, and nobody has a test for it -- it always just worked. Then a backend refactor renames the export controller, the link starts returning an HTML error page with a `.csv` name, and the first person to find out is the finance lead doing month-end close. Export links break silently because no human clicks them between releases. You want the suite to click this one every night.

### Solution

```gherkin
Feature: Reports export

  Scenario: CSV export downloads with the right name
    Given I am on "/reports/sales"
    When I download the file from the link "Download CSV"
    Then the downloaded file name should be "sales-2026-05.csv"
```

The download step does two things in the right order: it arms Playwright's `download` event listener first, *then* clicks the link by its visible text. That ordering matters -- if you clicked first and listened second, a fast server could finish the download before the listener existed, and you'd wait forever. Once the event fires, the bytes land in a Playwright-managed temp directory and webship-js holds the file handle for the rest of the scenario. The name assertion checks the suggested filename -- the one the server sent in `Content-Disposition` -- with an exact match. Every later download-related `Then` step refers to this same file until the next download replaces it.

**At scale:** almost every project in a portfolio has at least one export -- CSV reports, PDF invoices, ZIP backups -- and they are the least-clicked links in manual QA. Tag these scenarios `@downloads` across all your suites so one nightly CI job can run every export check in the portfolio. The exact-name assertion doubles as a contract test on the backend: when a server-side rename slips through code review, this is the test that names it before finance does.

---

## Recipe 388: Downloading by Partial File Name

### Problem
The first version of this test asserted the exact name `invoice-2026-05-31.pdf`. It passed for exactly one day. The server stamps every download with the current date, so any exact-name assertion is a time bomb -- it detonates at midnight, in the next month, or on the next billing cycle, and someone burns a morning "debugging" a test that was wrong from the start. You know the stable part of the name; assert only that.

### Solution

```gherkin
  Scenario: Invoice downloads with timestamped name
    Given I am on "/account/billing"
    When I download the file from the link "Download invoice"
    Then the downloaded file name should contain "invoice-"
```

`should contain` runs a substring match against the suggested filename instead of exact equality. The capture mechanics are identical to Recipe 387 -- listener armed before the click, file held on the scenario -- only the comparison relaxes. Use it whenever the server bakes dates, user IDs, or random tokens into names. The right discipline is to assert the parts the *contract* guarantees: the prefix that identifies the document type and, in a second assertion if you like, the extension.

**Pro Tip:** write this rule into your team's test style guide: "never pin a generated token in a filename assertion -- assert prefix and extension." It's the kind of trap every new hire falls into once. A one-line convention saves you the recurring midnight-rollover failure across every project that exports anything.

---

## Recipe 389: Downloading Directly from a URL

### Problem
Not every download hangs off an anchor you can click. Your dashboard builds the export URL in JavaScript and fires it from a button handler -- there is no `<a>` element, and the link-based step has nothing to grab. You tried hitting the URL with `curl` from the pipeline instead, and it returned a login page: the export endpoint needs the session that lives in the browser, not in your shell.

### Solution

```gherkin
  Scenario: Programmatic download from URL
    Given I am on "/reports"
    When I download the file from the URL "/api/exports/q1-summary.pdf"
    Then the downloaded file name should be "q1-summary.pdf"
```

This is the part teams miss: the URL variant fires the request *from inside the browser context*, so the session cookies, auth headers, and anything else the page carries ride along automatically. A `curl` or test-runner-side fetch would arrive anonymous and get redirected to login. The relative path resolves against your `launchUrl` -- the `LAUNCH_URL` environment variable, `http://localhost:8080` by default -- so the step doesn't care which environment it's pointed at. The response is captured as a download exactly as in Recipe 387, name assertion and all.

**At scale:** the relative-path-plus-`LAUNCH_URL` combination is what makes this step portable across a portfolio. The same scenario file runs against dev, staging, and production because the base URL is wired per pipeline as an environment variable -- nobody edits a config file per project. If you catch yourself hardcoding `https://staging.example.com/api/...` in a download step, stop: that scenario just became single-environment.

---

## Recipe 390: Asserting Downloaded File Contents

### Problem
The download works. The name is right, the size looks plausible, the test is green -- and the totals inside are from the wrong quarter, because a caching bug serves last period's report under this period's name. A name assertion can't see inside the file. When the file *is* the deliverable -- a sales report someone forwards to a director -- you need to assert what's written in it, not just that it arrived.

### Solution

```gherkin
  Scenario: Report contains the expected totals
    Given I am on "/reports/sales"
    When I download the file from the link "Download text report"
    Then the downloaded file should contain:
      """
      Total revenue: $12,450.00
      Total orders: 87
      """
```

The step reads the captured temp file as text and runs a substring match against it. One detail worth knowing: the docstring is a single string, newlines included, so those two lines must appear in the file *together, in that order, adjacent*. If the real report prints other lines between them, split into two separate content assertions instead of one block. This works for any text-based format -- plain text, CSV, JSON, XML. It does not work for binary formats: grepping a PDF's raw bytes will match garbage or nothing, and a ZIP needs the archive-aware steps below.

**Pro Tip:** for CSV exports, assert the header row. It's the schema contract -- column names and order -- and it breaks the moment someone reorders or renames a column, which is exactly the change that silently corrupts the spreadsheet your finance team imports. For PDFs, extract text in a hook (any PDF-to-text library) and assert against the extracted string, never the raw file.

---

## Recipe 391: Asserting ZIP Archive Contents

### Problem
Your "Download project" button packages a working directory into a ZIP. Six months in, someone refactors the packaging script and a whole folder quietly drops out of the archive. The suite stays green -- it only ever checked that *a* ZIP downloaded -- and the first report comes from a customer who unzipped an incomplete project. Archives are a blind spot: the test verified the envelope and never opened it.

### Solution

```gherkin
  Scenario: Project ZIP has the expected files
    Given I am on "/projects/42"
    When I download the file from the link "Download project"
    Then the downloaded file should be a zip archive containing the following files named:
      | README.md         |
      | src/index.js      |
      | tests/index.test  |
```

The step reads the captured ZIP in memory -- nothing is extracted to disk -- and walks the archive's entry list, checking that every row in the table exists as an entry path. Matching is exact: case-sensitive, full path from the archive root, forward slashes (ZIP entries use `/` regardless of the OS that built them). A missing entry fails with the name of the file that wasn't there, which is precisely the message you want at triage time.

**At scale:** treat the table as your packaging manifest. It is the written contract of what ships, and it lives in a feature file a product owner can actually read. Keep it next to the packaging code in the repo so the developer who adds a file to the export updates the expected list in the same pull request -- review catches the mismatch, not the customer.

---

## Recipe 392: ZIP Contents by Partial Name

### Problem
Your nightly backup ZIP stamps every entry with a timestamp: `db-snapshot-2026-06-11-0300.sql.gz`. You can't pin exact names without rewriting the test every night, but you absolutely need to know the snapshot is in there -- the day a refactor renames the prefix from `db-snapshot-` to `database-`, your restore runbook stops matching reality, and you find out during an actual restore. That is the single worst moment to learn your backup format changed.

### Solution

```gherkin
  Scenario: Backup ZIP contains a snapshot
    Given I am on "/admin/backups"
    When I download the file from the link "Download latest backup"
    Then the downloaded file should be a zip archive containing the following files partially named:
      | db-snapshot-      |
      | media-archive-    |
```

The `partially named` variant runs each table row as a substring match across every entry path in the archive; the assertion passes when at least one entry matches each row. Same in-memory unpacking as Recipe 391, looser comparison -- it's the archive version of the filename rule from Recipe 388: assert the stable prefix, never the generated token. Mind that "substring anywhere in the path" is genuinely anywhere: `db-snapshot-` also matches `old/db-snapshot-x`. Stack this step with the strict `named` variant in the same scenario when you want both guarantees -- "README.md exists exactly, and something matching `db-snapshot-` exists too."

---

## Recipe 393: Negative ZIP Assertion

### Problem
Here is a failure mode that ends up in incident reports: a developer adds a `secrets.env` to the build directory for local convenience, the export packager globs the whole directory, and the next public ZIP ships credentials to anyone with the download link. Nobody notices for months because every test asks "is the right stuff in the archive?" and none asks "is the wrong stuff out of it?". The positive assertions in Recipes 391-392 cannot catch this class of bug by design.

### Solution

```gherkin
  Scenario: Public export excludes secrets
    Given I am on "/projects/42"
    When I download the file from the link "Download public project"
    Then the downloaded file should be a zip archive not containing the following files partially named:
      | secrets.env       |
      | private/          |
      | .git/             |
```

The negation walks *every* entry path in the archive and fails the moment any listed substring appears in any of them. Directory rows like `private/` work because ZIP entries carry their full path -- `private/keys/prod.pem` contains the substring and trips the guard. The failure message names the offending entry, so triage is instant. Note the asymmetry with the positive step: `containing` needs one match per row to pass; `not containing` needs zero matches across the whole archive.

**At scale:** make this a standing security smoke test on every public artifact in the portfolio -- one shared scenario template, the same forbidden list (`secrets`, `.env`, `.git/`, `private/`, `node_modules/`), tagged `@security` so the nightly farm runs it everywhere. It costs seconds per project. The failure mode it prevents costs a disclosure notice. Maintain the forbidden list centrally with whoever owns security review, not per project, so a new pattern (say, `id_rsa`) lands in every suite at once.

---

## Recipe 394: Recording Video of a Test Run

### Problem
The booking flow fails once a week in CI and never on anyone's machine. The failure screenshot shows the end state -- a page without a confirmation number -- but not the *path* there: did the search return nothing, did the Reserve click land on the wrong row, did a modal eat the Confirm press? It's the Tuesday before release, the triage thread is twelve messages deep, and everyone is guessing. A 30-second video would end the argument in one viewing.

### Solution

```gherkin
Feature: Booking flow

  Scenario: Reservation completes end-to-end
    Given I start video recording
    And I am on "/booking"
    When I fill in "dates" with "2026-06-01 to 2026-06-05"
    And I press "Search"
    And I click "Reserve" in the "Sunset Suite" row
    And I press "Confirm"
    Then ".confirmation-number" should be visible within 10 seconds
    And I stop video recording
```

There are two ways to record in webship-js, and knowing which to reach for matters.

The first is the `WEBSHIP_VIDEO` environment variable, with four modes: `off`, `on`, `on-failure`, and `tag`. For the problem above -- "show me every failure" -- `on-failure` is the right answer, set once in CI with zero Gherkin changes:

```bash
WEBSHIP_VIDEO=on-failure npx cucumber-js
```

Videos land in `WEBSHIP_VIDEO_DIR` (default `./videos`) at 1280x720, named by the pattern `{datetime}.{feature_file}.{scenario}.{status}.{ext}` -- note `{status}` in the name, which is what lets your artifact step glob only the failures. In `tag` mode, only scenarios tagged `@video` record; in any mode, `@no-video` suppresses recording for a scenario and `@video` forces it on.

The second way is the in-scenario steps shown above -- and they come with a warning that belongs in bold. Playwright can only attach a recorder when a browser *context* is created, so `start video recording` and `stop video recording` rebuild the context mid-scenario. Rebuilding the context throws away everything in it: cookies, session, localStorage, the page you were on. Drop `Given I start video recording` after a login step and you will continue the scenario logged out, on a blank page, wondering why. That is exactly why this example puts it on the *first* line, before any state exists worth losing. Use the steps when you want recording scoped inside one scenario; use the env var for everything else.

**At scale:** set `WEBSHIP_VIDEO=on-failure` once in the shared CI configuration, not per project -- it's a pipeline concern, and it belongs next to the other env wiring (`LAUNCH_URL`, browser choice). Budget the storage: forty suites recording nightly add up even at 1280x720 webm, so let the `{status}` token do the work -- upload only `*.failed.*` as artifacts and purge the rest at the end of the job. A green run's video is a file nobody will ever watch.

---

## Recipe 395: Saving Video With a Custom Name

### Problem
You've finally reproduced ticket #1284 -- the checkout that misbehaves when "Apply" is clicked with an empty coupon field -- and you want to attach the recording to the ticket. But the artifacts directory holds forty videos named by the default datetime pattern, and "which timestamped file is the repro?" is a question you'll be asked three times this week. A recording you made on purpose deserves a name you chose on purpose.

### Solution

```gherkin
  Scenario: Repro for ticket #1284
    Given I start video recording
    And I am on "/checkout"
    When I click "Apply"
    And I save the current video as "bug-1284-empty-coupon.webm"
```

(The coupon field is deliberately never filled -- the empty submit *is* the bug being recorded.) The save step takes the recording in progress and writes it under the exact filename you give it, instead of the default `{datetime}.{feature_file}.{scenario}.{status}.{ext}` pattern. Because the name is stable across runs, re-running the repro overwrites the same file -- the artifact link you pasted into the ticket keeps pointing at the latest recording. Remember the context-rebuild rule from Recipe 394 still applies: `start video recording` is the first step here for a reason.

**Pro Tip:** put the ticket ID in the filename, always. Six weeks from now, nobody searches artifacts by date -- they search by ticket. `bug-1284-*.webm` is findable; `2026-06-09T03-14-22.checkout.feature` is archaeology.

---

## Recipe 396: Printing the Video Path

### Problem
First week with the suite, first local recording -- and then the hunt begins. The run finished, the video exists *somewhere*, and the generated filename is a long datetime-feature-scenario string you'd have to reconstruct by hand. Watching a new teammate `ls -lt` their way through directories to find a file the framework just wrote is a small, daily kind of friction. Ask the framework instead.

### Solution

```gherkin
  Scenario: Locate the video
    Given I start video recording
    And I am on "/"
    When I press "Search"
    Then print video path
```

The step resolves where the current recording will be written -- `WEBSHIP_VIDEO_DIR` (default `./videos`) plus the generated filename -- and logs the absolute path into the test output, right in the step stream where you're already looking. No directory spelunking, no reconstructing the `{datetime}.{feature_file}.{scenario}.{status}.{ext}` pattern in your head. It's the video sibling of `Then print current URL`: a debug step you drop in while investigating and delete before commit.

**Pro Tip:** no video player on your CI-image-matched dev container? Drag the `.webm` straight into a browser tab -- every Chromium and Firefox build plays webm natively.

---

## Practical Tips

**Downloads:**

- Always assert the file name first -- it's cheap and catches the "wrong file" class of bug.
- Use content assertions for text files and ZIP-content assertions for archives. Don't try to grep binary PDFs.
- For PDF content checks, render the PDF to text with a hook and assert against the text.

**Video:**

- Recording adds 5-10% overhead. Don't record every test by default -- set `WEBSHIP_VIDEO=on-failure` in CI and let the framework keep video only where it earns its disk space. Use `@video` to force recording on a scenario you're watching, `@no-video` to silence a noisy one.
- The filename pattern includes `{status}`, so your artifact-upload step can glob `*.failed.*` from `WEBSHIP_VIDEO_DIR` and skip green-run videos entirely.
- Never put `start video recording` mid-scenario after state you care about -- the context rebuild logs you out and blanks the page. First step or env var; those are the two safe positions.
- Combine with screenshots at key milestones for a hybrid evidence trail: stills for the report, video for the argument-settler.

Downloads test the file the user gets. Video records the path they took. Together, they cover both the artifact and the journey.

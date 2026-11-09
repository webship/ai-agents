# Chapter 24: Performance Optimization

---

A fast test suite gets run. A slow test suite gets ignored. When your suite takes forty-five minutes, developers push code without waiting for it. When it takes five minutes, it becomes a natural part of the workflow.

And on a multi-project team, slow is not just a culture problem -- it is a bill. Forty suites times forty-five minutes is thirty hours of runner time a night, paid in CI minutes or in queued pipelines. Every minute you shave off the suite pattern gets multiplied by every project that uses it.

This chapter shows you how to measure, identify bottlenecks, and optimize your test suite's performance.

---

## Recipe 216: Measuring Test Suite Performance

**Problem:** The suite "feels slow", so someone proposes buying bigger runners, someone else blames the staging server, and a third person wants to delete tests. Nobody has a number. Optimizing without measuring is how teams spend a sprint making the wrong thing 10% faster while three scenarios quietly burn four minutes each.

**Solution:**

```bash
# Time the entire suite
time npx cucumber-js

# Use the JSON formatter to get per-scenario timing
npx cucumber-js --format json:report.json

# Parse the report for slowest scenarios
node -e "
  const report = require('./report.json');
  const scenarios = [];
  report.forEach(feature => {
    feature.elements.forEach(scenario => {
      const duration = scenario.steps.reduce((sum, s) =>
        sum + (s.result?.duration || 0), 0);
      scenarios.push({
        name: scenario.name,
        duration: (duration / 1e9).toFixed(2) + 's',
        ms: duration / 1e6,
      });
    });
  });
  scenarios.sort((a, b) => b.ms - a.ms);
  console.log('Slowest scenarios:');
  scenarios.slice(0, 10).forEach((s, i) =>
    console.log(\`  \${i+1}. \${s.duration}  \${s.name}\`)
  );
"
```

**How It Works:** Cucumber's JSON formatter records a nanosecond duration on every step result, and the snippet above just sums them per scenario and sorts. webship-js already writes this report on every run -- the default profile sends JSON to `tests/reports/cucumber_report.json` -- so on most projects you can point the script at the file you already have instead of re-running anything. Suite time almost always follows a power law: the top ten scenarios carry a third of the runtime. That top-ten list is your work queue for the rest of this chapter -- and it tells you which fix applies, because a 90-second scenario is usually either a UI journey that belongs at the API level (Recipe 218) or a login repeated through the form forty times.

**At scale:** the same JSON feeds `bin/generate-reports.js`, which stamps reports with `--app-version`, `--env`, `--browser`, `--executed`, and arbitrary `--metadata Key=Value` pairs. Add a project identifier and collect the JSON files centrally, and you get the chart your delivery managers actually want: suite duration per project, per night, trending over a quarter. A suite that doubled its runtime in three months is a maintenance conversation you want to have *before* the nightly window overflows -- and the data makes it a planning item instead of an argument.

---

## Recipe 217: Optimizing Wait Times

**Problem:** The quarterly platform review lands and CI minutes are up forty percent -- the first proposal on the table is bigger runners. Before anyone spends money, profile a suite; the verdict is almost always the same: most of the wall-clock time is nobody doing anything. The test is waiting on the app, or worse, waiting on a number someone typed two years ago. Claim back the free seconds first -- the delays that exist for humans watching the screen, not for correctness.

**Audit your waits:**

```
Wait Type                │ Where it comes from         │ Can Optimize?
─────────────────────────┼─────────────────────────────┼──────────────────
smartSettle (navigation) │ 3000ms budget per page load │ No -- exits the
                         │                             │ moment page settles
smartSettle (after step) │ 1500ms budget per step      │ No -- same early exit
web-first assertions     │ poll 100ms, up to 5s        │ Only cost time when
                         │                             │ they are failing
minWaitTime.page         │ worldParameters (3000ms)    │ Lower only if the
                         │                             │ app is genuinely fast
before/after step delays │ worldParameters minWaitTime │ Set to 0 in CI
SLOW_MO                  │ 0 CI / 300 headless /       │ Keep 0 in CI; it
                         │ 800 headed (defaults)       │ exists for humans
wait Ns (static sleeps)  │ legacy suites               │ Delete -- never
                         │                             │ needed in 2.0
```

**Optimize cucumber.js for CI:**

```javascript
// cucumber.js - CI profile
module.exports = {
  ci: {
    worldParameters: {
      launchUrl: process.env.LAUNCH_URL || 'http://localhost:8080',
      minWaitTime: {
        before_scenario: 0,   // No delay
        after_scenario: 0,    // No delay
        before_step: 0,       // No delay
        after_step: 0,        // No delay
        page: 3000,           // Page load timeout only
      },
    },
  },
};
```

```bash
# CI run: no unnecessary delays
SLOW_MO=0 npx cucumber-js --profile ci
```

**How It Works:** the key distinction is budgets versus delays. smartSettle's numbers -- 3000ms per navigation, 1500ms after each state-changing step -- are *budgets*: it polls the AJAX counter, pending timers, and DOM-mutation timestamp every 50ms and returns the instant the page is quiet, so a fast page pays 300ms, not 3000ms. Cutting those buys you nothing on a healthy app and flake on a slow one. The `minWaitTime` values and `SLOW_MO`, by contrast, are *delays* -- unconditional pauses that exist so a human can watch -- and CI has no humans, so they go to zero. webship-js already defaults `SLOW_MO` to 0 in CI; the profile above makes the rest explicit. One thing not to do: do not reach for `WEBSHIP_AUTO_SETTLE=off` to save milliseconds. The settle cycle is what lets the suite run without a single static sleep -- switching it off trades a rounding error of speed for the entire flake category Chapter 21 exists to fix.

**At scale:** keep one `ci` profile shape in a shared template and configure differences through env vars (`LAUNCH_URL`, `BROWSER`, `SLOW_MO`) rather than editing `cucumber.js` per project. The day someone discovers a better timing setting, it rolls out by updating the template -- not by twenty pull requests, of which seventeen get merged and three are forgotten until their pipelines mysteriously run slow.

---

## Recipe 218: Moving Tests to the Right Level

**Problem:** Someone once wrote "invalid email shows an error" as a browser test, and it became the template. Now there are thirty validation scenarios, each one loading the registration page, filling the form, submitting, and waiting for the render -- twenty seconds apiece to verify decisions the server makes in milliseconds. Ten minutes of every run is spent driving a browser to ask questions that never needed one.

```
Before (all browser):              After (optimized):
┌──────────────────┐              ┌──────────────────┐
│ Browser test:    │  20s         │ API test:        │  0.5s
│ Go to /register  │              │ POST /api/users  │
│ Fill in name     │              │ with invalid     │
│ Fill in email    │              │ email            │
│ Press submit     │              │ Assert 422       │
│ See error        │              └──────────────────┘
└──────────────────┘

Save: 19.5 seconds per test
```

**Rule of thumb:** If the test does not need to verify visual behavior or user workflow, use an API test.

**How It Works:** the API steps live in the same webship-js suite and the same report -- `Given the API base URL is "..."`, `When I send a POST request to "/api/users" with values:`, `Then the API response code should be 422`, `Then the API response should contain "valid email"`. Under the hood they are axios calls: no browser launch, no page render, no settle cycle -- which is where the 20s-to-0.5s drop comes from. Migrate by class, not by scenario: all field validation moves down, all permission-by-role checks move down, and you *keep one* browser scenario per form to prove the error actually renders where a user can see it. The server saying 422 and the user seeing "Please enter a valid email" are two different behaviors; test each once, at its own level.

**At scale:** this is the cheapest CI capacity you will ever add. Thirty validation scenarios going from browser to API saves roughly ten minutes per project per run -- across twenty projects running nightly, that is more than three hours of runner time a night, bought with a refactor instead of a hardware budget. It also splits your reports along an ownership line a PMO can use: API failures mean the contract broke, browser failures mean the experience broke.

---

## Recipe 219: Parallel Execution Configuration

**Problem:** Your CI runner has eight cores, and your suite uses one of them -- scenario after scenario, single file, while the machine idles at 12%. The platform team sees a 40-minute pipeline and assumes the tests are heavy; the truth is they are sequential. Parallelism is the one optimization that needs no test changes at all -- *if* your scenarios are isolated.

**Solution:**

```bash
# Detect available CPU cores and use them
npx cucumber-js --parallel $(nproc)

# Or set a fixed number
npx cucumber-js --parallel 4
```

**package.json scripts:**

```json
{
  "scripts": {
    "test": "cucumber-js",
    "test:fast": "SLOW_MO=0 cucumber-js --parallel 4",
    "test:ci": "SLOW_MO=0 cucumber-js --parallel $(nproc) --profile ci"
  }
}
```

**Requirements:**
- Each scenario must be independent (no shared state)
- webship-js ensures this: each scenario gets its own browser
- Database operations should not conflict between parallel scenarios

**How It Works:** `--parallel N` makes Cucumber start N worker processes and hand scenarios to whichever worker frees up first. Each webship-js worker runs its own browser with its own context, so cookies, sessions, and localStorage are isolated by construction -- the only sharing left is whatever your scenarios do to the application's database, which is why Recipe 194's isolation work is the real prerequisite. Mind the memory math when picking N: every worker is a full browser, so `$(nproc)` on a 16-core runner can mean sixteen Chromium instances and an out-of-memory kill that looks like random flake. Start at 4, watch runner memory and the application server's response times, and raise it until one of them objects. And if going parallel surfaces failures, do not turn it off -- it just found real isolation bugs that sequential luck was hiding.

**At scale:** parallelism moves the bottleneck from the runner to the system under test -- twenty workers across several projects hammering one shared staging server will manufacture timeouts that read as test flake but are capacity. Coordinate with your platform team on the nightly picture: workers per suite times suites per window equals concurrent browser sessions staging must absorb. On metered CI, also note what parallel does and does not save: it cuts wall-clock minutes per pipeline (and developer waiting), while total compute stays roughly the same -- the win is feedback speed and nightly-window fit, not a smaller invoice.

---

## Performance Optimization Checklist

```
Quick Wins (minutes to implement):
  □ Set SLOW_MO=0 in CI
  □ Set all minWaitTime values to 0 in CI profile
  □ Remove unnecessary "wait Ns" steps
  □ Enable --parallel 4 (or more)

Medium Effort (hours):
  □ Convert data validation tests from browser to API
  □ Delete legacy "wait Ns" steps -- smartSettle and web-first
    assertions already cover them
  □ Remove redundant scenarios that test the same path
  □ Combine related assertions into fewer scenarios

Architecture Changes (days):
  □ Implement test data creation via API (faster than UI)
  □ Set up CI matrix for parallel browser testing
  □ Log in once and reuse it via the storage-state
    save/restore steps
  □ Build a smoke test suite for quick PR feedback
```

---

## Chapter Summary

- Measure before optimizing -- the JSON report you already generate knows where the time goes
- Human-facing delays (`SLOW_MO`, minWaitTime) go to 0 in CI; smartSettle budgets stay -- they exit early on their own
- Delete legacy static sleeps; never disable auto-settle to chase milliseconds
- Move data validation tests from browser to API level -- same suite, same report, 40x faster
- Parallel execution can cut wall-clock time by 75%+, if scenarios are isolated
- Saving and restoring storage state beats logging in through the form in every scenario
- Smoke test suite provides fast PR feedback
- A fast test suite gets used; a slow one gets ignored -- and across twenty projects, a slow one gets expensive

This concludes Part V: Troubleshooting and Maintenance.

Next: CI/CD Integration -- putting your test suite into production pipelines.

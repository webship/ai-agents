---
name: webship-issue-templates
description: Canonical Webship templates for issues and MRs/PRs — the drupal.org default issue summary template (HTML with ✅/❌/➖ Remaining tasks), the GitHub issue template (markdown adaptation), the Checkpoints checklist every MR/PR description must end with, and the in-repo `.gitlab/` and `.github/` issue + MR/PR templates every Webship project ships. Use when creating or updating an issue on drupal.org, git.drupalcode.org or GitHub, opening an MR on git.drupalcode.org or a PR on GitHub, adding the repo template files to a project branch, setting a drupal.org project's Custom issue summary template, or flipping ✅/❌/➖ marks as work progresses.
---

# Webship Issue & MR/PR Templates

Single source of truth for the templates used by every Webship agent when filing issues and opening MRs/PRs. Copy these verbatim — never improvise the structure; only fill in the content.

## Progress marks

| Mark | Meaning |
|------|---------|
| ✅ | Done |
| ❌ | Pending / not done yet |
| ➖ | Not applicable to this issue |

Update marks as work progresses — the same issue body is edited over its lifetime (❌ → ✅ when done; set ➖ once, when the item genuinely doesn't apply). In markdown checklists use `- [x]` for done and `- [ ]` for pending; drop nothing, reorder nothing.

## 1. Checkpoints — end of EVERY MR / PR description

GitHub PRs and GitLab (git.drupalcode.org) MRs end with this checklist, ticking only what is actually done at the time of writing:

```markdown
### Checkpoints:
- [x] File an issue
- [x] Addition/Change/Update/Fix
- [ ] Testing to ensure no regression
- [ ] Automated unit testing coverage
- [ ] Automated functional testing coverage
- [ ] UX/UI designer responsibilities
- [ ] Readability
- [ ] Accessibility
- [ ] Performance
- [ ] Security
- [ ] Developer Documentation
- [ ] User Guide Documentation
- [ ] Reviewed by human
- [ ] Code review by maintainers
- [ ] Full testing and approval
- [ ] Credit contributors
- [ ] Review with the product owner
- [ ] Release notes snippet
- [ ] Release
```

This is the same list that ships in a project's `.gitlab/merge_request_templates/default.md` (section 4), so an MR opened from the GitLab UI and one opened by an agent read identically.

## 2. drupal.org issues — default issue summary template (HTML)

Issue bodies on drupal.org are HTML (CKEditor). Always start from the full template; add content into the sections — never drop a section. Replace `N/A` only when there is a real change to describe.

> **First, check where the issue actually lives.** Some drupal.org projects have moved their queue onto **GitLab work items** — the issue URL is `https://git.drupalcode.org/project/<project>/-/work_items/<id>` and there is no `drupal.org/node/<nid>` behind it. **Work items render Markdown, not HTML.** Pasting the HTML template into one leaves raw `<h3>`/`<p>` tags on the page.
>
> For a GitLab work item, keep this exact template *structure* but write it in Markdown: `## Problem / Motivation`, fenced code blocks, and `- [ ]` / `- [x]` for Remaining tasks (GitLab turns those into real, tickable task items — see "drupal.org issues that live in GitLab work items" below). Everything else is unchanged: the commit format, the AI-disclosure line, the Checkpoints block, and the rule that **`Reviewed by a human` and `Code review by maintainers` are never ticked**.
>
> Rule of thumb: **only a classic drupal.org issue node is HTML. Everything on GitLab/GitHub — MR, PR, and work-item issue — is Markdown.**

```html
<h3 id="summary-problem-motivation">Problem/Motivation</h3>

<h4 id="summary-steps-reproduce">Steps to reproduce</h4>

<h3 id="summary-proposed-resolution">Proposed resolution</h3>

<h3 id="summary-remaining-tasks">Remaining tasks</h3>

<ul>
    <li>✅ File an issue</li>
    <li>❌ Addition/Change/Update/Fix</li>
    <li>❌ Testing to ensure no regression</li>
    <li>➖ Automated unit/functional testing coverage</li>
    <li>➖ Developer Documentation support</li>
    <li>➖ User Guide Documentation support</li>
    <li>➖ UX/UI designer responsibilities</li>
    <li>➖ Accessibility and Readability</li>
    <li>❌ Reviewed by a human</li>
    <li>❌ Code review by maintainers</li>
    <li>❌ Full testing and approval</li>
    <li>❌ Credit contributors</li>
    <li>❌ Review with the product owner</li>
    <li>❌ Update Release Notes</li>
    <li>❌ Release</li>
</ul>

<h3 id="summary-ui-changes">User interface changes</h3>

<ul>
    <li>N/A</li>
</ul>

<h3 id="summary-api-changes">API changes</h3>

<ul>
    <li>N/A</li>
</ul>

<h3 id="summary-data-model-changes">Data model changes</h3>

<ul>
    <li>N/A</li>
</ul>

<h3 id="summary-release-notes">Release notes snippet</h3>

<ul>
    <li>N/A</li>
</ul>
```

## 2b. drupal.org issues that live in GitLab work items — same template, Markdown

When the project's queue is on GitLab (`git.drupalcode.org/project/<project>/-/work_items/<id>`), use the SAME sections as the HTML template above, written in Markdown. The Remaining tasks / Checkpoints checkboxes become real GitLab task items.

````markdown
## Problem / Motivation

<what is broken, for whom, with the real error output in a fenced block>

## Steps to reproduce

1. …
2. …

## Proposed resolution

<what the fix does, citing the actual file/method>

## Remaining tasks

- [x] Addition/Change/Update/Fix to this project
- [ ] Reviewed by a human
- [ ] Code review by maintainers

## User interface changes

None.

## API changes

None.

## Data model changes

None.

## Release notes snippet

<one line, or None.>
````

`Reviewed by a human` and `Code review by maintainers` stay unticked — the AI never ticks them.

## 3. GitHub issues — Webship template (markdown)

On **GitHub**, do NOT reproduce the drupal.org summary template. No `### Remaining tasks` with ✅/❌ prose, and no `User interface changes` / `API changes` / `Data model changes` / `Release notes snippet` trailing sections. A GitHub issue is Problem/Motivation → Proposed resolution → the **Checkpoints** checklist (GitHub markdown checkboxes), the same checklist a PR ends with:

```markdown
### Problem/Motivation

#### Steps to reproduce

### Proposed resolution

### Checkpoints:
- [x] File an issue
- [ ] Addition/Change/Update/Fix
- [ ] Testing to ensure no regression
- [ ] Automated unit testing coverage
- [ ] Automated functional testing coverage
- [ ] UX/UI designer responsibilities
- [ ] Readability
- [ ] Accessibility
- [ ] Performance
- [ ] Security
- [ ] Developer Documentation
- [ ] User Guide Documentation
- [ ] Reviewed by human
- [ ] Code review by maintainers
- [ ] Full testing and approval
- [ ] Credit contributors
- [ ] Review with the product owner
- [ ] Release notes snippet
- [ ] Release
```

Tick only what is genuinely done (`- [x]`), leave the rest `- [ ]`; never tick `Reviewed by a human` or `Code review by maintainers`. Items that don't apply: strike them (`- [ ] ~Automated unit/functional testing coverage~`) — GitHub checkboxes have no ➖. The ✅/❌/➖ marks and the UI/API/Data-model/Release-notes sections are **drupal.org only** (the HTML template above); they never appear in a GitHub issue or PR.

## 4. Project repo templates — `.gitlab/` on git.drupalcode.org

Every Webship-maintained project on git.drupalcode.org carries these two files, on **every branch that has an active `.x-dev` release**, so the GitLab UI pre-fills the right shape for anyone opening a work item or an MR. GitLab renders both as **Markdown** — never HTML.

### `.gitlab/issue_templates/default.md`

```markdown
### Problem/Motivation

#### Steps to reproduce

### Proposed resolution


### Remaining tasks:
- [x] File an issue
- [ ] Addition/Change/Update/Fix
- [ ] Testing to ensure no regression
- [ ] Automated unit testing coverage
- [ ] Automated functional testing coverage
- [ ] UX/UI designer responsibilities
- [ ] Readability
- [ ] Accessibility
- [ ] Performance
- [ ] Security
- [ ] Developer Documentation
- [ ] User Guide Documentation
- [ ] Reviewed by human
- [ ] Code review by maintainers
- [ ] Full testing and approval
- [ ] Credit contributors
- [ ] Review with the product owner
- [ ] Release notes snippet
- [ ] Release

### API changes
* N/A

### Data model changes
* N/A

### Release notes snippet
* N/A
```

### `.gitlab/merge_request_templates/default.md`

The Checkpoints checklist from section 1, as the whole file:

```markdown
### Checkpoints:
- [x] File an issue
- [x] Addition/Change/Update/Fix
- [ ] Testing to ensure no regression
- [ ] Automated unit testing coverage
- [ ] Automated functional testing coverage
- [ ] UX/UI designer responsibilities
- [ ] Readability
- [ ] Accessibility
- [ ] Performance
- [ ] Security
- [ ] Developer Documentation
- [ ] User Guide Documentation
- [ ] Reviewed by human
- [ ] Code review by maintainers
- [ ] Full testing and approval
- [ ] Credit contributors
- [ ] Review with the product owner
- [ ] Release notes snippet
- [ ] Release
```

## 5. Project repo templates — `.github/` on github.com

The GitHub mirrors carry the same two shapes, in GitHub's own locations. Both are **Markdown**. Per section 3, a GitHub issue never carries the drupal.org ✅/❌/➖ marks or the UI/API/data-model/release-notes sections — it ends with **Checkpoints**.

### `.github/ISSUE_TEMPLATE/default.md`

```markdown
---
name: Default
about: Report a problem or propose a change
title: ''
labels: ''
assignees: ''
---

### Problem/Motivation

#### Steps to reproduce

### Proposed resolution

### Checkpoints:
- [x] File an issue
- [ ] Addition/Change/Update/Fix
- [ ] Testing to ensure no regression
- [ ] Automated unit testing coverage
- [ ] Automated functional testing coverage
- [ ] UX/UI designer responsibilities
- [ ] Readability
- [ ] Accessibility
- [ ] Performance
- [ ] Security
- [ ] Developer Documentation
- [ ] User Guide Documentation
- [ ] Reviewed by human
- [ ] Code review by maintainers
- [ ] Full testing and approval
- [ ] Credit contributors
- [ ] Review with the product owner
- [ ] Release notes snippet
- [ ] Release
```

### `.github/pull_request_template.md`

Identical body to `.gitlab/merge_request_templates/default.md` (the Checkpoints checklist from section 1) — one checklist, both hosts.

## 6. drupal.org project settings — Custom issue summary template

On the project node's edit form (drupal.org → the project → **Edit** → *Issue settings* → **Custom issue summary template**), paste the HTML from section 2 verbatim. That field is HTML because a classic drupal.org issue node is HTML; it is the only place in this file where HTML is correct alongside section 2.

**Which format, where — the one rule that decides it:**

| Destination | Format |
|-------------|--------|
| Classic drupal.org issue node (`drupal.org/node/<nid>`) and the project's Custom issue summary template field | **HTML** |
| GitLab work item on git.drupalcode.org, `.gitlab/` templates, GitLab MR description | **Markdown** |
| GitHub issue, GitHub PR description | **Markdown** |

HTML pasted into a GitLab or GitHub body leaves raw tags on the page; Markdown pasted into a drupal.org issue node shows its `###` and backticks literally.

## Usage rules

1. **Issue before work** — file the issue first; tick `File an issue about this project` the moment it exists (✅ on drupal.org, `- [x]` on GitHub). On GitHub the issue carries the **Checkpoints** checklist, not a drupal.org-style ✅/❌ Remaining-tasks section.
2. **MR/PR after issue** — reference the issue (`Closes #<id>` on GitHub; issue link on drupal.org) and end the description with the Checkpoints checklist (section 1).
3. **Keep marks honest** — tick/flip only what actually happened; "Reviewed by a human" stays ❌ / unticked until a human reviewed it.
4. **Commit format on drupal.org issue forks** — `{type}: #{issue-id} Summary` per <https://www.drupal.org/node/3586390> (`fix` `feat` `ci` `docs` `perf` `refactor` `test` `task` `revert`, no `chore`); MR title identical.
5. **AI disclosure** — per the [Drupal AI policy](https://www.drupal.org/docs/develop/issues/issue-procedures-and-etiquette/policy-on-the-use-of-ai-when-contributing-to-drupal), add `AI-Generated: Yes (<what>; reviewed by <contributor>.)` to commits and MR/PR descriptions.
6. **Contributor identity** — ask the user for the name/email to commit and file as (default `git config user.name` / `user.email`).
7. **Keep the format** — free-form content from a user or calling agent gets merged INTO these templates, never used instead of them; dropping the template requires the user's explicit confirmation.
8. **Worked examples** — for patch work on webship-patches / drupal-core-patches, follow [`references/webship-patches-examples.md`](references/webship-patches-examples.md): the Add/Remove/Change title grammar, the patch-file vs composer.json PR split, and the immutable-patch re-roll rule, all with real issue/PR numbers.
9. **One issue + one PR per fix** — never bundle multiple patches/fixes into one issue or one PR; each change gets its own dedicated issue and its own PR/MR so every review thread tells one clean story. If an issue/PR ends up mixing several, close it and re-create separate single-purpose ones.
10. **Reuse vs. new MR** — if a drupal.org / git.drupalcode.org issue already has an MR we can push to: a **small** change (minor edit, reroll, tweak) → commit to that existing MR; a **big** change (substantially different approach/diff) → open a **new** MR. No accessible MR (or the existing one is another contributor's fork we can't push to) → open our own issue-fork MR. Never hijack someone else's MR.
11. **Byte-exact copies for committing** — when adding the repo templates to a project branch, copy the files from [`references/repo-templates/`](references/repo-templates/) rather than retyping them from the fenced blocks above (`gitlab/issue_templates/default.md`, `gitlab/merge_request_templates/default.md`, `github/ISSUE_TEMPLATE/default.md`, `github/pull_request_template.md`, and `drupal-org-custom-issue-summary-template.html` for the project-settings field). The fenced blocks in this file are the documentation; those files are the artifact.
12. **Never tick the human-review flags** — the AI must never check `Reviewed by a human` or `Code review by maintainers`; they stay `- [ ]` / ❌ until the human reviewer sets them.

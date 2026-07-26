# Webship AI Agents

Reusable [Claude Code](https://claude.com/claude-code) **agents** and **skills** for building, testing, and
releasing the [Webship](https://www.drupal.org/project/webship) Drupal distribution and its ecosystem.

Drop the `.claude/` folder into a project (or merge into `~/.claude/`) to make these agents and skills
available to Claude Code.

## Layout

```
.claude/
├── agents/   # Claude Code sub-agent definitions (one .md per agent)
└── skills/   # Claude Code skills (one folder per skill, each with SKILL.md)
```

## Agents

**Release workflow (Drupal ~11.4.0 cycle)**
- `webship-11.0.x-release` — capstone agent for the whole Webship 11.0.x release workflow (modules, theme,
  profile/distribution rollup, project template).
- `webship-drupal-module-release` — cut a tag-only release of a single web\* module on drupal.org /
  git.drupalcode.org (+ github mirror).
- `webship-drupal-theme-release` — the theme counterpart.
- `webship-patches-release` — cut and manage releases of the `webship/patches` Composer plugin on
  github.com (auto-publishes to Packagist via webhook).
- `webship-drupal-patches-release` — the release counterpart for the `webship/drupal-patches` core-patch
  metapackage (one branch per Drupal core major.minor).

**Issues, patches & MR/PR lifecycle**
- `webship-issue-template` — create/maintain GitLab work items using the Webship `.gitlab` issue templates.
- `webship-drupal-issue-manager` — drupal.org issue + issue-fork bookkeeping.
- `webship-mr-pr-manager` — the single MR/PR lifecycle gateway across GitHub PRs and GitLab /
  git.drupalcode.org MRs (issue forks, Checkpoints checklist, commit-type titles). Never merges.
- `webship-patches` — install/configure the `webship/patches` Composer plugin (allowlist, wildcard
  ignore, `patches-ignore`), author and re-roll patches, diagnose patch failures.
- `webship-drupal-patches` — maintain the `webship/drupal-patches` Composer metapackage: curate a
  core-minor patch set, add a new Drupal core minor branch, wire it into `webship/patches`.

**Testing**
- `agent-webship-js` — automated browser testing with [webship-js](https://www.npmjs.com/package/webship-js)
  (Playwright + Cucumber-js): scaffold, author `.feature` files, run, and report.
- `webship-ai-agent` — webship-js BDD authoring / running / fixing loop.

**Workspace tooling** (`webship-workspace-*`) — build, back up, and maintain the folders of the
`~/workspace` Drupal development workspace: agents, components, demos, dev, docs, forked, libraries, modules,
products, profiles, projects, recipes, sandboxes, skills, test, themes.

## Skills

- `webship-patches` — the `webship/patches` Composer plugin controls (allowlist, wildcard ignore,
  `patches-ignore`) and curated contrib patches.
- `webship-drupal-patches` — the `webship/drupal-patches` metapackage, one branch per Drupal core
  major.minor.
- `patch-management` — generic, non-Webship patch creation/re-roll mechanics for any Drupal project.
- `webship-issue-templates` — the Webship issue-summary + Checkpoints templates (with saved copies of
  the Drupal AI policy and commit-types reference).
- `webship-mr-pr-manager` — the MR/PR lifecycle conventions (description shape, Checkpoints last,
  commit-type titles).
- `webship-js-init`, `webship-js-create`, `webship-js-run`, `webship-js-audit`, `webship-js-steps` —
  the webship-js BDD testing skills (scaffold a suite, author scenarios, run it, audit results, and manage
  step definitions).

## Notes

- Agents reference credentials by **file path only** (e.g. a git.drupalcode.org token at
  `~/.config/drupalcode/gitlab-token`) — no secrets are stored in this repo.
- Passwords that appear in test fixtures (e.g. `dD.123123ddd`) are throwaway local **DDEV test** credentials.
- `webship/patches` and `webship/drupal-patches` are renamed continuations of the earlier
  `webship/webship-patches` and `webship/drupal-core-patches` packages: fresh tag lines (`11.0.0` and
  `11.4.0` respectively), shorter names, and (for `webship/patches`) 3-segment never-move release tags
  instead of the predecessor's 4-segment scheme.

## License

GPL-2.0-or-later

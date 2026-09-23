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
- `webship-11-0-x-release` — capstone agent for the whole Webship 11.0.x release workflow (modules, theme,
  profile/distribution rollup, project template).
- `webship-drupal-module-release` — cut a tag-only release of a single web\* module on drupal.org /
  git.drupalcode.org (+ github mirror).
- `webship-drupal-theme-release` — the theme counterpart.
- `webship-patches-release` — cut and manage releases of the `webship/patches` Composer plugin on
  github.com (auto-publishes to Packagist via webhook).
- `webship-drupal-patches-release` — the release counterpart for the `webship/drupal-patches` core-patch
  metapackage (one branch per Drupal core major.minor).
- `cucumber_starter-1-0-x-release` — cuts releases of the `cucumber_starter` tag-only recipe package on
  drupal.org / git.drupalcode.org (+ github mirror), `1.0.x` only, release notes in the
  Added/Changed/Fixed style rather than the flat module/theme bullet form.
- `webship_project-12-0-x-release`, `webships_project-12-0-x-release`, `cucumber_project-12-0-x-release`,
  `website-12-0-x-release` — cut releases of the four tag-only `drupal/*_project` templates, `12.0.x`
  only, each gated on a green pipeline and Packagist packaging landing before the release node is
  published.

**Issues, patches & MR/PR lifecycle**
- `drupal-issue-manager` — issues in a **drupal.org node queue** (HTML bodies, no write API, browser only).
- `drupalcode-issue-manager` — issues that live as **GitLab work items** on git.drupalcode.org (Markdown,
  GitLab REST API), and the five `.gitlab/issue_templates/*.md` a project ships.
- `drupalcode-mr-manager` — the **git.drupalcode.org** merge-request lifecycle: issue forks, the Commits
  API, the `gitlab-ci-local` green gate, Checkpoints checklist, commit-type titles. Never merges.
- `github-pr-manager` — the **github.com** issue and pull-request lifecycle, including the patch-repo PR
  rules for `webship/patches` and `webship/drupal-patches`. Never merges.

  Which one to use is decided by the host, and by where the project's issues actually live — open a real
  issue URL and look, rather than inferring it from the project name.
- `webship-patches` — install/configure the `webship/patches` Composer plugin (allowlist, wildcard
  ignore, `patches-ignore`), author and re-roll patches, diagnose patch failures.
- `webship-drupal-patches` — maintain the `webship/drupal-patches` Composer metapackage: curate a
  core-minor patch set, add a new Drupal core minor branch, wire it into `webship/patches`.

**Testing**
- `agent-webship-js` — automated browser testing with [webship-js](https://www.npmjs.com/package/webship-js)
  (Playwright + Cucumber-js): scaffold, author `.feature` files, run, and report.
- `webship-ai-agent` — webship-js BDD authoring / running / fixing loop.

**Site templates**
- `drupal-site-template-creator` — scaffold a new Drupal recipe-based site template end to end: repo,
  clone-and-rename, branch, tracking issue, README, and the first dev release. Product-neutral, so it
  works for any recipe-based template, not only the Webship ones.
- `website_starter-1-0-x-site-template-manager` — maintains the shipped `website_starter` template.
- `webship_starter-1-0-x-site-template-manager` — maintains the shipped `webship_starter` template.
- `webship_portal-1-0-x-site-template-manager` — maintains the shipped `webship_portal` template.
- `cucumber_starter-1-0-x-site-template-manager` — maintains the shipped `cucumber_starter` template, the
  default site template of the `cucumber` install profile.
- `webapi_starter-2-0-x-site-template-manager` — maintains the shipped `webapi_starter` template.
- `webships_starter-2-0-x-site-template-manager` — maintains the shipped `webships_starter` template.

**Project templates** — `composer create-project` starters (`"type": "project"`), each scaffolding a
codebase that requires a distribution/installer and (for `website`, `webships_project`) a choice of site
template. Not recipes, not site templates — see the managers above for that side of the work.
- `webship_project-12-0-x-manager` / `webship_project-12-0-x-release` — maintain and release
  `webship_project`, `12.0.x` only.
- `webships_project-12-0-x-manager` / `webships_project-12-0-x-release` — maintain and release
  `webships_project`, `12.0.x` only.
- `cucumber_project-12-0-x-manager` / `cucumber_project-12-0-x-release` — maintain and release
  `cucumber_project`, `12.0.x` only.
- `website-12-0-x-manager` / `website-12-0-x-release` — maintain and release the `website` project
  template, `12.0.x` only.

**Front end & design systems**
- `drupal-themer` — the master themer. Decides where a change belongs (component, display
  configuration, theme tokens, or the framework's own classes), then orchestrates the sub-agents
  below, spawning one component builder per component.
- `drupal-sdc-component-builder` — authors exactly one Single Directory Component at a time.
- `drupal-page-assembler` — assembles pages from components that already exist.
- `ui-suite-uikit-themer` — the themer for the `ui_suite_uikit` theme (UIkit).
- `webtheme-themer` — the themer for the `webtheme` theme.

  Verification is delegated to `drupal-frontend-render-verifier` and token work to
  `drupal-design-token-mapper`, rather than duplicated here.

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
- `drupal-mr-manager` — the MR/PR lifecycle conventions shared by `github-pr-manager` and
  `drupalcode-mr-manager` (description shape, Checkpoints last, commit-type titles).
- `webship-js-init`, `webship-js-create`, `webship-js-run`, `webship-js-audit`, `webship-js-steps` —
  the webship-js BDD testing skills (scaffold a suite, author scenarios, run it, audit results, and manage
  step definitions).
- `drupal-site-template-prove` — what "proven" means for a site template: counted install assertions
  across both supported bases and all three install paths, not a finished install you looked at.
- `drupal-site-template-catalog` — the site template catalogue: package, repository, the exact
  non-interactive build invocation, and the recipes each template applies.

## Notes

- [`.claude/agents/RULES.md`](.claude/agents/RULES.md) is the single source of truth for the rules
  every agent follows — disclosure, evidence, destructive actions, and what must never appear in
  public content. Agents point at it rather than restating it.
- Agents reference credentials by **file path only** (e.g. a git.drupalcode.org token at
  `~/.config/drupalcode/gitlab-token`) — no secrets are stored in this repo.
- No contributor is hardcoded anywhere: identity resolves at run time, and worked examples use
  placeholders (`<your-gitlab-username>`) rather than real accounts.
- Passwords that appear in test fixtures (e.g. `dD.123123ddd`) are throwaway local **DDEV test** credentials.
- `webship/patches` and `webship/drupal-patches` are renamed continuations of the earlier
  `webship/webship-patches` and `webship/drupal-core-patches` packages: fresh tag lines (`11.0.0` and
  `11.4.0` respectively), shorter names, and (for `webship/patches`) 3-segment never-move release tags
  instead of the predecessor's 4-segment scheme.

## License

GPL-2.0-or-later

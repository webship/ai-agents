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
- `webship-issue-template` — create/maintain GitLab work items using the Webship `.gitlab` issue templates.
- `webship-mr-manager` — open issue-fork merge requests using the Webship `.gitlab` MR templates (never merges).

**Testing**
- `agent-webship-js` — automated browser testing with [webship-js](https://www.npmjs.com/package/webship-js)
  (Playwright + Cucumber-js): scaffold, author `.feature` files, run, and report.
- `webship-ai-agent` — webship-js BDD authoring / running / fixing loop.

**Workspace tooling** (`webship-workspace-*`) — build, back up, and maintain the folders of the
`~/workspace` Drupal development workspace: agents, components, demos, dev, docs, forked, libraries, modules,
products, profiles, projects, recipes, sandboxes, skills, test, themes.

## Skills

- `webship-js-init`, `webship-js-create`, `webship-js-run`, `webship-js-audit`, `webship-js-steps` —
  the webship-js BDD testing skills (scaffold a suite, author scenarios, run it, audit results, and manage
  step definitions).

## Notes

- Agents reference credentials by **file path only** (e.g. a git.drupalcode.org token at
  `~/.config/drupalcode/gitlab-token`) — no secrets are stored in this repo.
- Passwords that appear in test fixtures (e.g. `dD.123123ddd`) are throwaway local **DDEV test** credentials.

## License

GPL-2.0-or-later

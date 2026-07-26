---
name: webship-drupal-patches
description: Maintain webship/drupal-patches (renamed from the predecessor webship/drupal-core-patches) — the Composer metapackage holding Webship's curated Drupal core patches, with one git branch per Drupal core major.minor (currently 11.4.x and 12.0.x, plus a flat "patches" file-store branch). Covers the branch-per-core-minor scheme, curating a core-minor set, immutable dated patch files, Packagist-safe never-move release tags (semver within the minor), and wiring it into webship/patches (require range + allowlist). Use when adding a Drupal core minor branch, re-rolling/curating a core patch set, releasing the package, or debugging why a core patch is not applied.
---

# Drupal Core Patches (webship/drupal-patches)

A Composer **metapackage** that isolates Webship's curated **Drupal core** patches from the Webship line, so Webship can upgrade to the latest Drupal core while keeping the right patch set per core version. It is required by `webship/patches`. It is **NOT a Composer plugin** — never list it in `config.allow-plugins`; only in `extra.composer-patches.allowed-dependency-patches`.

## Branch per Drupal core major.minor

| Branch    | Drupal core | Notes                                    |
|-----------|-------------|-------------------------------------------|
| `11.4.x`  | `~11.4.0`   | default branch, curated core patch set    |
| `12.0.x`  | `~12.0.0`   | forward-compat placeholder (no patches)   |
| `patches` | n/a         | flat `.patch` file store (append-only)    |

The predecessor `webship/drupal-core-patches` carried the older core-minor branches (`10.4.x` … `11.3.x`); `webship/drupal-patches` starts at `11.4.x` and adds new minors going forward.

Each core-minor branch:

```json
{
  "require": { "drupal/core": "~<minor>.0" },
  "extra": { "patches": { "drupal/core": { "<description>": "<raw patches-branch URL>" } } }
}
```

The `require drupal/core ~<minor>.0` binds the release to that core minor, so Composer only picks the release whose minor matches the installed core. Patch URLs point at the `patches` branch:
`https://raw.githubusercontent.com/webship/drupal-patches/refs/heads/patches/<file>`.

## How the right set is chosen

The consumer (`webship/patches`) requires the broad range (`~11 || ~12`). Because each drupal-patches release pins `drupal/core ~<minor>.0`, Composer resolves the one release matching the installed core → the site automatically gets the patch set for ITS Drupal core.

## Curating a core-minor set

1. Identify the core patches the target minor needs (from the closest existing branch, or from upstream issues for a brand-new minor).
2. Download those patch files into the `patches` branch; point the new branch's composer URLs at them.
3. Create `<minor>.x` off the nearest branch; set `require drupal/core ~<minor>.0` + the set; copy docs/LICENSE/PR-template. See [docs/adding-a-core-version.md](https://github.com/webship/drupal-patches/blob/11.4.x/docs/adding-a-core-version.md) in-repo.
4. Tag `<minor>.0`.

## Immutable patch files — never re-roll in place

A `.patch` on the `patches` branch is immutable. To re-roll (new core minor, updated MR, corrected diff), add a **NEW** file with **today's** date — `drupal-core--$(date +%Y-%m-%d)--<issue>--mr-<N>.patch` — never the old date/filename, never overwrite; point the branch composer URLs at the new file and leave the old one so pinned releases keep resolving. Only exception: same-day content fix on a file created today, before any release referenced it.

Materialize every drupal.org core MR into a static, timestamped, standard-named file
(`curl -sL "<mr-url>.diff" -o "patches/drupal-core--$(date +%Y-%m-%d)--<issue>--mr-<n>.patch"`) — never
a raw MR URL (URLs drift, break checksums). Verify the diff starts with `diff --git`, not the
git.drupalcode.org bot-challenge `<!DOCTYPE html>`; if HTML, `git diff origin/<target>...<mrBranch> > patches/<file>.patch`.

## Patching history (`CHANGELOG.md`) — add the entry with the patch

When you add a patch to the patch list (`composer.json` `extra.patches`) in `webship/patches` or `webship/drupal-patches` — or change / remove one — add a matching entry under that branch's `## [Unreleased]` section of `CHANGELOG.md` **in the same change**. Each release branch keeps one newest-first `CHANGELOG.md`; the Unreleased section stages what the next release regenerates. Never ship a patch change without its Unreleased line.

## Releasing (CRITICAL)

- **Release title = tag only.** A GitHub Release on `webship/patches` or `webship/drupal-patches` MUST use the **tag as its exact title/name** (e.g. `11.0.0`, `11.4.0`) — no description suffix, no "Webship Patches …" / "Drupal core … patch set" text in the title. Any human-readable summary goes in the release **notes/body**, never the title.
- Tag **3-segment semver within the minor** — `11.4.0`, then `11.4.1`, `11.4.2`, … on `11.4.x`. The tag line restarted at `11.4.0` with the rename; the predecessor's 4-segment tags stay in the old repository. See the in-repo [docs/releasing.md](https://github.com/webship/drupal-patches/blob/11.4.x/docs/releasing.md).
- **Never move a tag** — Packagist rejects moved tags ("The last update failed"). Re-release an already-tagged commit by cutting a NEW tag with the last segment bumped; never `git tag -f`.
- **Packagist auto-publishes via the GitHub webhook** — verify via the p2 metadata (`https://repo.packagist.org/p2/webship/drupal-patches.json`); the `patches` branch has no composer.json, so Packagist ignores it.
- A future-core branch (`12.0.x`) stays a forward-compat placeholder: `require drupal/core ~<minor>.0` with an **empty** `extra.patches."drupal/core"` until patches are re-rolled for that core.
- **Tick `Release` after the tag.** After you cut / publish a release tag for `webship/patches` or `webship/drupal-patches`, tick the `- [x] Release` checkpoint on the associated **issue AND PR**, adding a link to the released tag (e.g. `Released in https://github.com/webship/drupal-patches/releases/tag/11.4.0`). `Release` is a factual post-release tick done by the releaser — this is **allowed**; it does **not** let the AI tick `Reviewed by a human` or `Code review by maintainers` (human-only).

## Wiring into webship/patches

`webship/patches` **requires** `webship/drupal-patches` (`~11 || ~12`) and its plugin allowlists the package so the core patches apply — via the constant `Webship\Patches\Plugin\PatchesPlugin::DEFAULT_ALLOWED_DEPENDENCY_PATCHES` (`['webship/patches', 'webship/drupal-patches']`) used by the v2 `FilteredDependencies` resolver.

## Issue / PR titles + shared-file rule

Same grammar as webship-patches — `<Action> a patch for the <Target> on <ref>` (Add / Remove / Change / Update / Revert -) — Target is usually **`Drupal Core`**, the core minor is the context. Copy the upstream drupal.org issue's FULL title verbatim (no paraphrase, no `fix:`/`task:` prefix, no `(#id)` duplication). One materialized core `.patch` on the file-store branch, referenced from each core-minor branch that needs it — never duplicated. Every PR ends with the Checkpoints checklist. Read the branch `CHANGELOG.md` before adding/removing/changing a patch.

## Related skills & agents

- Paired agent: **webship-drupal-patches** — the full sub-agent form of this skill.
- Release agent: **webship-drupal-patches-release** — cutting and publishing releases.
- **webship-patches** skill + agent — the plugin that requires this metapackage; contrib (non-core) patches live there.
- **patch-management** skill — generic, non-Webship cweagans/composer-patches declare/apply/create/re-roll mechanics.
- **webship-mr-pr-manager** skill + agent — opens/maintains the patch PRs; **webship-issue-templates** skill for the issue + Checkpoints templates.

---
name: webship-patches
description: Apply and manage patches using cweagans/composer-patches (v2) for Drupal projects, and use the webship/patches Composer plugin's allowlist, wildcard ignore, and patches-ignore extensions on top of v2. Use when applying patches to Drupal core or contrib, configuring composer.json patch blocks, handling patch failures, or integrating with Webship patches.
---

# Composer Patches

Apply and manage patches using `cweagans/composer-patches` for Drupal projects. Covers v2 and the `webship/patches` plugin that wraps it with extra controls.

## Prerequisites

- Composer 2.x.
- `cweagans/composer-patches: ~2.0`, declared in `require` and allowed in `config.allow-plugins`.

## Enable patching

```json
{
  "config": {
    "allow-plugins": {
      "cweagans/composer-patches": true
    }
  },
  "extra": {
    "enable-patching": true,
    "composer-exit-on-patch-failure": true
  }
}
```

`composer-exit-on-patch-failure: true` is the safer default — silent patch failures hide regressions until production.

## Declare patches

### Inline (`extra.patches`)

```json
{
  "extra": {
    "patches": {
      "drupal/module_name": {
        "Issue #1234567: Description of the fix": "patches/module_name--2026-05-10--1234567--mr-42.patch"
      }
    }
  }
}
```

### External patches-file

```json
{
  "extra": {
    "patches-file": "composer.patches.json"
  }
}
```

`composer.patches.json` follows the same `{ "<pkg>": { "<description>": "<url>" } }` shape as the inline `patches` block.

### Patch sources

- Local files: `patches/module--fix.patch`
- Drupal.org issue files: `https://www.drupal.org/files/issues/<date>/<file>.patch`
- GitLab MR raw: `https://git.drupalcode.org/project/<name>/-/merge_requests/<n>.patch`
- GitHub commit / PR: `https://github.com/<org>/<repo>/commit/<sha>.patch`

Prefer **local, timestamped files** for production. Raw MR URLs change as commits are pushed to the MR and break Composer checksums mid-install.

## Filename convention (Webship)

```
[package]--[YYYY-MM-DD]--[issue number]--[mr number].patch
```

Examples:

- `drupal-core--2026-05-10--3539178--mr-12890.patch`
- `ctools--2026-05-10--3572317--mr-85.patch`
- `ui_patterns--2023-12-17--3409221-3--mr-21.patch`

## v2-only controls

### `ignore-dependency-patches` — exact match (upstream v2)

```json
{
  "extra": {
    "composer-patches": {
      "ignore-dependency-patches": ["drupal/specific_module"]
    }
  }
}
```

Upstream v2 only matches by exact package name.

### Patch levels

```json
{
  "extra": {
    "patches": {
      "drupal/core": {
        "-p2": true
      }
    }
  }
}
```

## Extra controls from `webship/patches`

`webship/patches` (renamed from the predecessor `webship/webship-patches`) is a Composer plugin (`type: composer-plugin`) layered on top of v2. It restores v1-style behaviors and adds wildcards.

### `allowed-dependency-patches` (allowlist, default-deny)

```json
{
  "extra": {
    "composer-patches": {
      "allowed-dependency-patches": ["webship/patches", "webship/drupal-patches"]
    }
  }
}
```

Only listed packages may contribute dependency-declared patches. Default: `["webship/patches", "webship/drupal-patches"]` (constant `Webship\Patches\Plugin\PatchesPlugin::DEFAULT_ALLOWED_DEPENDENCY_PATCHES`). Net effect: only Webship-curated patches and your project's own `extra.patches` apply — stale third-party `.patch` URLs in unrelated contrib modules are skipped.

### Wildcard `ignore-dependency-patches`

```json
{
  "extra": {
    "composer-patches": {
      "ignore-dependency-patches": ["drupal/*", "another/specific-package"]
    }
  }
}
```

`fnmatch`-style globbing. Applied after the allowlist.

### `patches-ignore` (v1-style, restored on v2)

Exclude one specific patch URL declared by a given dependency:

```json
{
  "extra": {
    "patches-ignore": {
      "webship/patches": {
        "drupal/core": {
          "Issue description": "https://patch-url.patch"
        }
      }
    }
  }
}
```

Schema: `{ "<source-pkg>": { "<target-pkg>": { "<description>": "<url>" } } }`. Matching is done by URL — the description string is informational. A flat array of URLs (`{ "<source-pkg>": { "<target-pkg>": ["<url>", ...] } }`) is also accepted.

## No plugin Composer commands (dropped in the rename)

The predecessor `webship/webship-patches` shipped `webship-patches:cleanup:patches` (`web-ccup`) and
`webship-patches:cleanup:patches-file` (`web-ccupf`) Composer commands that rewrote remote MR URLs in
`composer.json` to local timestamped files. **`webship/patches` dropped both** — materialize an MR diff
manually instead:

```bash
curl -sL "<mr-url>.diff" -o "patches/<pkg>--$(date +%Y-%m-%d)--<issue>--mr-<n>.patch"
head -1 "patches/<file>.patch"   # must be "diff --git", not "<!DOCTYPE html>"
```

## Examples

### Add a local patch

```json
{
  "extra": {
    "patches": {
      "drupal/paragraphs": {
        "Fix paragraph duplication issue": "patches/paragraphs--2026-05-10--duplication-fix.patch"
      }
    }
  }
}
```

### Apply and update

```bash
composer update drupal/paragraphs --with-dependencies
```

### Create a patch from a modified contrib checkout

```bash
cd web/modules/contrib/<module>
git diff > ../../../../patches/<module>--$(date +%Y-%m-%d)--fix.patch
```

### Verify before committing

```bash
git apply --check patches/<file>.patch
```

## Handling patch failures

- **Already applied** (upstream merged the fix): remove from `extra.patches`, or — for a patch declared by `webship/patches` — add to `patches-ignore`.
- **Patch conflicts**: re-roll against the new module version, rename the file with a fresh `YYYY-MM-DD`, update the entry in `extra.patches`.
- **Stale third-party URL aborts install**: rely on the default allowlist (`["webship/patches", "webship/drupal-patches"]`) or add a wildcard `ignore-dependency-patches` (e.g. `drupal/*`).
- **`Failed to open stream` on fresh `composer create-project`**: a downstream plugin set `extra.plugin-modifies-downloads` or `extra.plugin-modifies-install-path` and got promoted to early activation before `drupal/core` was extracted. Drop those flags — `webship/patches` uses late activation (POST_PACKAGE_INSTALL of itself) on purpose.

## See also

- Agent: `webship-patches` — end-to-end Webship patches workflows for the supported line.
- Agent: `webship-patches-release` — cutting and publishing `webship/patches` releases.
- Skill: `webship-drupal-patches` — the sibling Drupal-core-patches metapackage.
- [Webship Patches repo](https://github.com/webship/patches)
- [cweagans/composer-patches](https://github.com/cweagans/composer-patches)

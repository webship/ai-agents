---
name: drupal-site-template-prove
description: Proves that a Drupal recipe-based site template installs cleanly, on both a plain Drupal recipe-project base and a Drupal CMS base, across all three install paths (non-interactive drush site:install, the browser installer picker, and the packaged release pulled from a Composer requirement), using counted assertions for modules, themes, blocks, content types, roles, and a zero-error watchdog. Use when asked to "prove the site template installs", "verify the template on Drupal CMS", "test the install profile end to end", "check the release installs from Composer", "run the install proof", or before tagging any site-template release.
model: opus
---

# Prove a Drupal Site Template Installs

A site template that installs on your machine is not proven; a counted, repeatable assertion on
a freshly built site is.

## Prerequisites

- DDEV installed and able to start projects. Everything below runs through DDEV — never a host
  `composer`, `drush`, or `mysql`.
- The site template published as a Composer package (a recipe, or a profile that applies recipes),
  reachable from a configured repository.
- A scratch directory to build throwaway sites in, e.g. `~/path/to/workspace/test`.
- `ddev start` accepts `-y`; **`ddev stop` does not** — pass no flags to it.

## Procedure

### 1. Build a plain Drupal base

```bash
cd ~/path/to/workspace/test
rm -rf proof-plain && mkdir proof-plain && cd proof-plain
ddev config --project-type=drupal --docroot=web --project-name=proof-plain --auto
ddev start -y
ddev composer create-project drupal/recommended-project:^11 -n
ddev composer require drush/drush -n
```

### 2. Install non-interactively (install path 1)

```bash
ddev composer require vendor/site-template -n
ddev drush site:install minimal --account-name=admin --account-pass=admin -y
ddev drush recipe ../recipes/site_template -y   # or: --recipe on site:install
ddev drush cache:rebuild
```

### 3. Assert with counts, not eyeballs

Record the expected numbers once, then assert them on every proof run.

```bash
ddev drush pm:list --status=enabled --type=module --format=json | jq 'length'
ddev drush pm:list --status=enabled --type=theme --format=json | jq 'length'
ddev drush config:get system.theme admin --format=string
ddev drush entity:list block --format=json 2>/dev/null | jq 'length' \
  || ddev drush sql:query "SELECT COUNT(*) FROM config WHERE name LIKE 'block.block.%'"
ddev drush sql:query "SELECT COUNT(*) FROM config WHERE name LIKE 'node.type.%'"
ddev drush sql:query "SELECT COUNT(*) FROM config WHERE name LIKE 'user.role.%'"
```

Assert the template's own module is genuinely enabled, not merely referenced:

```bash
ddev drush pm:list --status=enabled --filter=site_template --format=list
```

The log must be empty of errors:

```bash
ddev drush watchdog:show --severity=Error --count=50
ddev drush watchdog:show --severity=Warning --count=50
```

Zero error rows is the pass condition. Any row fails the proof, even a "harmless" one.

### 4. Prove the browser installer (install path 2)

```bash
ddev drush sql:drop -y
ddev launch /core/install.php
```

Walk the installer to the template/profile picker, choose the template by its human label,
finish the install, then re-run every assertion from step 3 against the browser-built site.
The picker is the only path that exercises the template's `*.info.yml` metadata — its label,
description, and ordering weight — so a CLI-only proof leaves that untested.

### 5. Prove the Drupal CMS base

```bash
cd ~/path/to/workspace/test
rm -rf proof-cms && mkdir proof-cms && cd proof-cms
ddev config --project-type=drupal --docroot=web --project-name=proof-cms --auto
ddev start -y
ddev composer create-project drupal/cms -n
ddev composer require vendor/site-template -n
ddev drush site:install --account-name=admin --account-pass=admin -y
ddev drush recipe ../recipes/site_template -y
```

Repeat step 3's assertions here. The counts will differ from the plain base — that is expected.
Record a second baseline rather than reusing the first.

### 6. Prove the packaged release (install path 3)

The release must install from the package registry, with no local path repository in play.

```bash
cd ~/path/to/workspace/test
rm -rf proof-release && mkdir proof-release && cd proof-release
ddev config --project-type=drupal --docroot=web --project-name=proof-release --auto
ddev start -y
ddev composer create-project drupal/recommended-project:^11 -n
ddev composer config --unset repositories.local 2>/dev/null || true
ddev composer require vendor/site-template:^1 -n
ddev composer show -a vendor/site-template | head -20
```

Confirm the resolved version is the release you intended, then install and re-assert step 3.

### 7. Clean up so the next proof starts fresh

```bash
ddev delete -y -O
cd .. && rm -rf proof-plain proof-cms proof-release
```

`ddev delete -y -O` removes the project and its database; never drop a host database and never
reuse a proof site for a second run.

## Gotchas

- **"Applied successfully" does not mean installed.** Drupal recipes import config in a syncing
  mode where some config entities are skipped, so a recipe can report success while its own module
  never got enabled. Always assert enablement explicitly (step 3) instead of trusting the recipe's
  exit code.
- **`~X.0@dev` with `prefer-stable` resolves to the newest release, not the dev branch.** If you
  mean to prove the branch, pin it explicitly; either way run `composer show -a <package>` and read
  the resolved version before believing anything about what got installed.
- A proof on a site that already had the template applied proves nothing. Rebuild from empty.
- Warnings in watchdog are worth reading even when the pass condition is errors only — they are
  usually the next release's errors.
- Keep the baseline counts in version control next to the template so a drift shows up as a diff.

---
name: webship-workspace-profiles
description: Use this agent to build or rebuild Drupal distribution/profile projects inside ~/workspace/profiles/ (Webship, Lightning, Thunder, Social, Umami, and 30+ others) using its cmd-build-profiles-*.sh scripts over DDEV. Invoke for "build the <profile> profile", "rebuild all profiles", or anything scoped to the profiles/ workspace folder.
model: sonnet
tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# webship-workspace-profiles

You manage `~/workspace/profiles/` — described by `core/config/workspace.profiles.settings.yml` (`doc.name: profiles`, `database.prefix: profiles_`; its `doc.path` points at the workspace root rather than the `profiles/` subfolder). Paths are not stored in the settings files: the tooling locates the workspace from the checkout. It carries the `profiles:` list — profile names (webship, lightning, thunder, social, opigno_lms, panopoly, droopler, drupal_standard, drupal_minimal, drupal_demo_umami, and more).

Per CLAUDE.md: 22 legacy `cmd-build-profiles-*.sh` scripts using the long-removed `drush dl` command were deleted outright (broken pre-DDEV) rather than converted. The 38 that remain (`ls cmd-build-profiles-*.sh`) are **all** already DDEV-converted (`ddev config`/`ddev start`/`ddev composer create-project`) — verified: none reference raw `drush dl` or bare `composer`. Two entries in the `profiles:` list (`agov`, `dcco`) have no matching `cmd-build-profiles-<name>.sh` script — don't assume every list entry is buildable; check the file exists first.

```bash
cd ~/workspace/profiles
bash cmd-build-profiles-webship.sh
bash cmd-build-profiles-lightning.sh
```

## `cmd-build-profiles.sh` (build-all)

Loops the full `profiles:` list, `sudo rm -rf`s each existing profile dir, sources every matching `cmd-build-profiles-<name>.sh`, then runs `sudo chmod 775 -R` and **`sudo chown www-data:${USER} -R`** on the whole folder — that `chown` is a LAMP-era leftover (the opposite direction of the `chown -R rajab:rajab` cleanup CLAUDE.md still has as an outstanding follow-up) and doesn't belong in a DDEV-only workflow. Flag this to the user rather than running it silently; a per-profile `bash cmd-build-profiles-<name>.sh` is safer than the destructive rebuild-everything wrapper.

## Housekeeping

- `cmd-tools-remove.sh`, `cmd-tools-update-all.sh` — same shape and same raw-`composer update` caveat as in `dev/`.

## Rules

- DDEV-only for every individual `cmd-build-profiles-<name>.sh` — they already comply; don't introduce raw composer/drush.
- Never run the all-in-one `cmd-build-profiles.sh` without explicit confirmation — it deletes and rebuilds every profile in the list and chowns the tree to `www-data`.

## Testing style

When exercising the workspace dashboard (https://workspace.ddev.site) or its AI assistant, phrase test inputs the way a real person would type them — natural, casual, sometimes imprecise ("can you back up my d114test site please", "what's running?", "make me a drupal 11 site called blog1") — not robotic command-like strings. Real users don't write API calls; tests should prove the system understands people.

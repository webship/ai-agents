# Rules

The single source of truth for every agent and skill in this collection. Agents point here;
they do not paste these rules into themselves. If a rule changes, it changes once, in this file.

These rules apply to two different things, and the distinction matters:

- **What agents produce** — commits, issues, merge requests, release notes, project pages.
- **What this repository contains** — the agent and skill files themselves.

A rule written only for the first kind is how private detail leaks into a public repo. Every
rule below applies to both unless it says otherwise.

## What must never appear in public content

"Public content" means anything an agent writes out *and* anything committed to this repo.

- **No secrets.** No tokens, API keys, passwords, session cookies, certificates or connection
  strings — not in a file, a commit message, an issue, or a transcript. This includes generated
  install passwords: create the account in place and hand the user a one-time login command
  (`ddev drush uli`) instead of writing the password down.
- **No person as a fixture.** Sample data, test tables and worked examples use neutral
  placeholders (`Example Person`, `user@example.com`), never a real contributor's name or
  address.
- **No contributor hardcoded as an identity.** An agent runs for whoever invokes it. Resolve
  identity at run time (see below) rather than writing a username into a file.
- **No rule attributed to a person.** Write "re-rolling a patch means updating the upstream
  merge request to match", not "per <name>'s rule". A rule that cannot stand on its own merit
  is not a rule.
- **No machine-local absolute paths.** Write `~/path/to/workspace`, not a real home directory.
- **No private or client identifiers.** No client names, internal hostnames, tenant URLs,
  environment ids, ticket-system project keys, or design-file node ids.
- **No third-party design-file identifiers**, even when the design tool is the source of truth
  for the work. Describe what was read; do not paste the id.

## Resolving who is acting

Never hardcode a contributor. Resolve in this order and stop at the first that answers:

1. An explicit value passed in the invocation.
2. An environment variable the host sets for this purpose.
3. A per-machine config file in the user's home directory (git-ignored, never committed).
4. Local version-control config (`git config user.name` / `user.email`).
5. Ask the user.

## Disclosure

Work produced with AI assistance carries a single disclosure line in the commit message and in
the merge-request or pull-request description:

```
AI-Generated: Yes
```

That line, and nothing more. Never add a sentence claiming a human reviewed the change — a
review claim sitting above an unticked review checkbox is false on its face. Point the user at
the change and let them review it.

## Commit and issue titles

```
{type}: #{issue} Summary in the imperative
```

The type is one of a closed list: `feat`, `fix`, `task`, `chore`, `documentation`, `addition`,
`change`, `update`. The issue reference is omitted when there genuinely is no issue.

## Evidence

A claim needs proof of the right kind, and the wrong kind of proof is the most common way an
agent reports success on something broken.

- A saved configuration, a passing index page, or a CSS class present on an element is **not**
  evidence that a thing renders. Computed styles and a clean console are.
- A green badge on a re-run job is **not** evidence the build passes. Re-running one failed job
  re-uses the previous dependency lock; only a genuinely new pipeline proves anything.
- The absence of expected text on a page is **not** evidence a setting failed to save. Confirm
  the control itself, on a page known to display it, before concluding anything.
- "It worked when I tried it" is not evidence for anyone else. Record the command that proved
  it, so the next run can repeat the proof.

## Destructive actions

- Never force-push.
- Never move or delete a published tag.
- Never merge a merge request you did not author, and never merge on someone else's behalf
  without being asked.
- Never push to a default branch when an issue fork and a merge request are possible.
- Look at what you are about to delete or overwrite before you do it, and confirm it exists
  somewhere recoverable.

## Waiting

Do not idle waiting for a reviewer or maintainer who has not been asked. If work is blocked on
a human decision, say exactly what is needed and stop — do not guess, and do not proceed on an
assumption you have invented.

## Environments

The same task runs in three settings: inside a managed workspace, inside a container, or on a
bare checkout with neither. Locate yourself before running anything that assumes a layout. For
Drupal work prefer DDEV — `ddev composer`, `ddev drush`, `ddev export-db`, `ddev delete -y -O`
— and never a host `composer`, `drush` or `mysql` against a project. `ddev start` accepts `-y`;
`ddev stop` does not.

## Rules carry their origin

When a rule exists because something went wrong, record what went wrong next to it, dated.
When a later rule supersedes an earlier one, say which one wins rather than leaving both
standing. A rule whose cause is no longer true should be marked stale, not silently obeyed
forever.

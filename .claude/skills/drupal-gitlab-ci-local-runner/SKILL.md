---
name: drupal-gitlab-ci-local-runner
description: Run a Drupal project's GitLab CI pipeline on your own machine with gitlab-ci-local before pushing to git.drupalcode.org — install it, wire the drupal/gitlab_templates remote variables, run all jobs or one job, read failures, and hold a green gate before asking for a merge. Use before pushing to an issue-fork branch, when a pipeline fails and you need to reproduce it locally, when a job cannot run locally, or when adding a slim .gitlab-ci-local.yml to a project.
---

# Running the pipeline locally (gitlab-ci-local)

**The green gate.** Before pushing to an issue-fork branch, run what the pipeline can run locally with
`gitlab-ci-local`, and push only when it is green. A failing, errored or masked job blocks the push —
fix it first. Keep every job `allow_failure: false`, and never mask a check with `|| true`.

For what the pipeline should *contain*, see the **`drupal-gitlab-ci-templates`** skill.

## 1. Install

```bash
npm install -g gitlab-ci-local
gitlab-ci-local --version
docker info        # every job runs in a container
```

## 2. Wire the Drupal templates

A project that includes `drupal/gitlab_templates` needs the template's own variables resolved, or
every job fails on an unset `_GITLAB_TEMPLATES_REPO`. Wrap it once, as the upstream docs do — save as
`~/.local/bin/drupal-ci-local` and `chmod +x`:

```bash
#!/bin/bash
gitlab-ci-local \
  --remote-variables git@git.drupal.org:project/gitlab_templates=includes/include.drupalci.variables.yml=main \
  --variable="_GITLAB_TEMPLATES_REPO=project/gitlab_templates" \
  "$@"
```

Upstream notes that cloning over `https://` "does not work completely without issues"; the SSH URL
above is deliberate. Have your key loaded in `ssh-agent` first.

## 3. Everyday commands

```bash
gitlab-ci-local --list                       # what jobs exist, and their stage
gitlab-ci-local phpcs                        # one job by name
gitlab-ci-local --file .gitlab-ci-local.yml  # the slim, locally reproducible pipeline
gitlab-ci-local --shell-isolation            # isolate shells so artifacts behave correctly
gitlab-ci-local --variable DRUPAL_CORE=11.4.8
```

Artifacts and logs land in `.gitlab-ci-local/` — **gitignore it.**

## 4. The blocker you may hit first

Running the full `drupal/gitlab_templates` pipeline can crash before any job runs:

```
Error attempting to evaluate the following rules:
  - if: '"8.3" == "8.5" && ("$CORE_PHP_MAX" =~ '/^\"8.3"$/' …
```

That is [firecow/gitlab-ci-local#1893](https://github.com/firecow/gitlab-ci-local/issues/1893): a
rule with a `$variable` inside an `=~` regex breaks rule evaluation, and the Drupal templates use that
pattern. Until it is fixed upstream:

- The slim `.gitlab-ci-local.yml` (§6) is the runnable local path for a templates-based project.
- Template-only jobs gate on the **remote** pipeline; say so in the report rather than calling the
  slim run full coverage.
- Do not burn time on the crash: it is not the project, SSH, Docker or your variables.

## 5. Read the failure properly

- **`composer` fails → everything fails.** Later jobs consume its artifact. Fix it first.
- **PHPUnit runs against the composer-installed copy, not your working tree.** The installed copy is at
  `.gitlab-ci-local/artifacts/composer` — diff it against your tree when a fix "does nothing".
- **Browser jobs need container networking help.** Expect them to be awkward locally; a dry run of the
  suite (`--dry-run`) still proves the step definitions load and how a split shares the scenarios.
- **`execution took longer than 30m0s`** on the remote pipeline is the runner's limit, not a failing
  test — see "Runner time limit" in `drupal-gitlab-ci-templates`.
- **A job that cannot run locally is not a pass.** Say so and gate on the remote pipeline.
- Check that `--list` actually contains the job you think you ran.

## 6. Ship a slim local pipeline

Heavy jobs (a full Drupal build, browser tests) are painful locally. Commit a second file holding only
the reproducible subset:

```yaml
# .gitlab-ci-local.yml — the validate-stage jobs that reproduce locally.
# Left out: composer, phpunit and the browser suite, which gate on the remote pipeline.
# Usage:
#   gitlab-ci-local --file .gitlab-ci-local.yml          # all jobs
#   gitlab-ci-local --file .gitlab-ci-local.yml phpcs    # one job
stages:
  - validate

phpcs:
  stage: validate
  image: php:8.4-cli
  before_script:
    - apt-get update -y > /dev/null && apt-get install -y -qq git unzip > /dev/null
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer global config --no-plugins allow-plugins.dealerdirect/phpcodesniffer-composer-installer true
    - composer global require --no-interaction --no-progress drupal/coder:^8.3 dealerdirect/phpcodesniffer-composer-installer:^1
    - export PATH="$HOME/.composer/vendor/bin:$PATH"
  script:
    - phpcs --standard=.phpcs.xml
  allow_failure: false
```

## 7. The gate, as a procedure

1. `gitlab-ci-local --list` — confirm the job set.
2. Run the runnable jobs (the slim file if the project ships one).
3. All green → push. Anything red, errored or unrunnable → stop, fix, re-run, or name it as gated on
   the remote pipeline.
4. Report honestly: which jobs ran, which could not run locally, and the exact failing output.

## References

- `gitlab-ci-local`: <https://github.com/firecow/gitlab-ci-local>
- Drupal's instructions: <https://project.pages.drupalcode.org/gitlab_templates/info/test-locally/>
- Companion skill: `drupal-gitlab-ci-templates`

---
name: drupal-site-template-catalog
description: Maintains the catalogue of available Drupal site templates — a markdown table mapping each template to its Composer package, repository, exact non-interactive build/install invocation, and the recipes it applies — and defines the four-step procedure for adding a new template to it. Use when asked "what site templates are available", "which template should I build", "how do I install template X", "add this template to the catalogue", "list the site templates", or when a new recipe-based site template is published and needs to become discoverable.
model: opus
---

# Drupal Site Template Catalogue

The catalogue is the contract — a template nobody can find and build is not shipped.

## Prerequisites

- DDEV installed; every build below runs through it. Never a host `composer`, `drush`, or `mysql`.
- The catalogue file itself lives in the repository that owns the templates, e.g.
  `~/path/to/workspace/docs/site-templates.md`. This skill defines its shape.
- Each listed template is published as a Composer package from a reachable repository.
- `ddev start` accepts `-y`; **`ddev stop` does not**.

## The catalogue table

Every row answers four questions: what do I require, where does it live, what exactly do I type to
get a working site, and which recipes did that apply. Keep the columns in this order.

| Template | Composer package | Repository | Non-interactive build + install | Recipes applied |
| --- | --- | --- | --- | --- |
| Corporate Site | `vendor/corporate-template` | `https://git.example.org/vendor/corporate-template` | see *Corporate Site* below | `corporate_base`, `corporate_content` |
| Editorial Newsroom | `vendor/newsroom-template` | `https://git.example.org/vendor/newsroom-template` | see *Editorial Newsroom* below | `newsroom_base`, `media_library_setup` |
| Documentation Portal | `vendor/docs-template` | `https://git.example.org/vendor/docs-template` | see *Documentation Portal* below | `docs_base`, `search_setup` |

### Corporate Site

```bash
mkdir corporate && cd corporate
ddev config --project-type=drupal --docroot=web --project-name=corporate --auto
ddev start -y
ddev composer create-project drupal/recommended-project:^11 -n
ddev composer require vendor/corporate-template:^1 drush/drush -n
ddev drush site:install minimal --account-name=admin --account-pass=admin -y
ddev drush recipe ../recipes/corporate_base -y
ddev drush recipe ../recipes/corporate_content -y
ddev drush cache:rebuild
ddev launch
```

### Editorial Newsroom

```bash
mkdir newsroom && cd newsroom
ddev config --project-type=drupal --docroot=web --project-name=newsroom --auto
ddev start -y
ddev composer create-project drupal/cms -n
ddev composer require vendor/newsroom-template:^1 -n
ddev drush site:install --account-name=admin --account-pass=admin -y
ddev drush recipe ../recipes/newsroom_base -y
ddev drush recipe ../recipes/media_library_setup -y
ddev drush cache:rebuild
```

### Documentation Portal

```bash
mkdir docs-portal && cd docs-portal
ddev config --project-type=drupal --docroot=web --project-name=docs-portal --auto
ddev start -y
ddev composer create-project drupal/recommended-project:^11 -n
ddev composer require vendor/docs-template:^1 drush/drush -n
ddev drush site:install minimal --account-name=admin --account-pass=admin -y
ddev drush recipe ../recipes/docs_base -y
ddev drush recipe ../recipes/search_setup -y
```

Tear any of them down the same way:

```bash
ddev delete -y -O
```

## Adding a template to the catalogue

### 1. Confirm the package resolves from its registry

```bash
ddev composer show -a vendor/new-template
```

Read the resolved version and the listed repository. A template that only installs from a local
path repository is not catalogueable yet — publish it first.

### 2. Capture the exact invocation by running it

Build a throwaway site with the commands you intend to publish, copy-pasting nothing you have not
executed. Record the base (plain recipe-project or Drupal CMS), every `ddev composer require`, and
every `ddev drush recipe` call in order.

```bash
ddev drush recipe:list 2>/dev/null || ls ../recipes
```

### 3. Add the row and its command block

Add one table row plus a matching `### <Template>` section with the fenced block from step 2. The
row's *Recipes applied* column must list exactly the recipes the block applies, in the same order.
Then tear the scratch site down with `ddev delete -y -O`.

### 4. The install proof must stay green

Run the `drupal-site-template-prove` skill against the new template before the catalogue change is
merged, and again on every release. A catalogue row is a promise that the commands in it produce a
working site; the proof is what keeps that promise honest.

## Gotchas

- Recipe application order is load-bearing. A row that lists recipes alphabetically instead of in
  applied order will mislead whoever copies it.
- `~X.0@dev` with `prefer-stable` resolves to the newest release, not the dev branch — state the
  constraint you actually tested, and verify with `composer show -a <package>`.
- Do not mix bases in one row. A template supported on both a plain recipe-project and Drupal CMS
  gets two command blocks, not one block with a comment.
- Keep the table product-neutral: any Drupal site builder should be able to append a row without
  first adopting a particular distribution.
- A catalogue entry whose proof is failing is worse than no entry. Remove or mark the row rather
  than leaving a build that does not work.

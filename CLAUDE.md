# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Composer-managed **Drupal CMS 2.1.1** site (on **Drupal core 11.3**), scaffolded from the
**Byte site template** and developed locally with **DDEV** (`type: drupal11`, docroot `web/`).
The project name "referenzen-api" is German for "references."

Current state: still close to a stock Byte-template install. There are **no custom modules or
themes yet** (`web/modules/custom` and `web/themes/custom` don't exist), and **no site config has
been exported** — [config/sync](config/sync) holds only `.gitkeep`. Treat the first `cex` and the
first custom module/theme as net-new scaffolding, not edits to existing code.

`AGENTS.md` (this directory) and [web/themes/contrib/byte_theme/AGENTS.md](web/themes/contrib/byte_theme/AGENTS.md)
contain authoritative guidance that this file summarizes — read them before non-trivial work.

## Project goal (where this is headed)

This Drupal install is a **headless / decoupled content hub**, not a public-facing website. Two
deliverables drive the work:

1. **References API.** The maintainer runs several **Astro** sites (freelance + agency) that today
   each hardcode their own reference / case-study entries. Drupal becomes the single source of truth:
   a `Referenz` content model exposed over an API (Drupal core ships **JSON:API**) that the Astro
   sites fetch **at build time** (SSG) instead of hardcoding.
2. **Automated CV / résumé generation.** Produce a polished **CV as a PDF** automatically — driven
   from the same reference / skill / experience data, and a natural fit for the bundled AI stack
   (drafting / summarization) plus a Drupal PDF renderer (e.g. `entity_print`).

Implications for how to work here:

- **The public front-end is Astro, not Drupal.** byte_theme / Canvas matter only for the *editorial*
  experience — don't invest in public-facing theming. Keep content presentation-agnostic and the API
  payload clean.
- **The API contract is the product.** Field machine names, JSON:API resource shapes, and any custom
  endpoints are consumed by external builds; treat changes as breaking and communicate/version them.
- **None of this is built yet.** The `Referenz` content type, its API exposure, and the CV/PDF
  generator are all to-do. Expect to scaffold a custom module (e.g. `web/modules/custom/referenzen`)
  and to choose a PDF approach. Build the content model and field names deliberately — they become
  the external contract the moment an Astro site consumes them.

## Local development (DDEV)

All PHP/Drush/Composer commands run **inside DDEV**. Run from the project root:

```bash
ddev start                       # boot the environment (runs composer install + recipe-unpack on post-start)
ddev launch                      # open the site in a browser
ddev composer install            # install PHP dependencies
ddev composer require drupal/x   # add a contrib module/theme (then enable + rebuild, see below)
ddev drush status                # site status
ddev drush uli                   # one-time admin login link
ddev drush cr                    # cache:rebuild — run after most config/code changes
ddev drush updb -y               # apply DB updates after enabling/updating modules
```

Add a module: `ddev composer require drupal/<project>` → `ddev drush pm:enable -y <machine_name>` → `ddev drush cr`.

## Configuration management

[config/sync](config/sync) is the canonical config directory — the scaffold appends logic
([assets/append-default.settings.php.txt](assets/append-default.settings.php.txt)) that points
`$settings['config_sync_directory']` at `../config/sync`. This is the unit of version-controlled
site state.

```bash
ddev drush cex -y    # export site config TO config/sync (commit the result)
ddev drush cim -y    # import config FROM config/sync into the site
```

After changing anything in the admin UI that should persist, run `cex` and commit the diff.

## Recipes define the site

[recipes/](recipes) holds Drupal recipes (Composer-installed, **gitignored**, unpacked on
`ddev start` via `composer drupal:recipe-unpack`). Recipes — not hand-written config — are how this
site is composed:

- **[recipes/byte/recipe.yml](recipes/byte/recipe.yml)** is the active site template. It installs
  `byte_theme` (and sets it as default), sets the front page to `/home` (built in Canvas), and pulls
  in the `drupal_cms_*` building-block recipes (`admin_ui`, `forms`, `media`, `search`, `seo_*`,
  `anti_spam`, `authentication`, `content_type_base`, `privacy_basic`, …).
- **recipes/haven** is an alternative site template (non-profit oriented; ships blog/projects/people).
- **[recipes/drupal_cms_ai/recipe.yml](recipes/drupal_cms_ai/recipe.yml)** wires the AI stack.

Apply an additional recipe to a running site with `ddev drush recipe recipes/<name>`. Site templates
(byte/haven) are normally applied once at install time by the Drupal CMS installer.

## Page building: Canvas + Byte theme

> Editorial/admin-side only. Per the project goal, public rendering is Astro — this stack is for the
> content-editing experience, not the delivered front-end.

- **Canvas** (`drupal/canvas`) is the visual page builder. Content bundles carry a
  `field_component_tree` field that stores the Canvas component tree; the home page and landing pages
  are Canvas-built. The byte recipe disables a long list of Canvas blocks that aren't useful for page
  building.
- **byte_theme** ([web/themes/contrib/byte_theme](web/themes/contrib/byte_theme)) is a **Tailwind CSS
  v4** theme built from **SDC** (single-directory components, in `components/`) and **CVA**
  (`html_cva`, from `drupal/cva`) for class variants. Note: Canvas *code components* are currently
  incompatible with Tailwind themes — see the theme README before creating one.

### Theme work

byte_theme has a build step (Tailwind compiles `src/main.css` → `build/main.min.css`). From the theme
directory (run via `ddev exec` or on the host if Node is installed; `node_modules` is not committed):

```bash
npm install
npm run dev      # watch + rebuild during development
npm run build    # one-off production build
npm run format   # prettier (Twig + Tailwind class sorting)
```

[web/themes/contrib/byte_theme/AGENTS.md](web/themes/contrib/byte_theme/AGENTS.md) defines strict Twig
conventions that must be followed when editing components — the essentials:

- Use **CVA (`html_cva`)** for all conditional classes; never put `{% if %}` inside a `class` (or any)
  attribute. Variant keys are `yes`/`no` strings, not booleans. Use arrays for long class lists.
- Includes must use `with only` (tag) or `with_context: false` (function), and pass only props the
  component actually declares.
- Always `npm run format` and `npm run build` after changes.

**Caveat:** byte_theme lives under `contrib/` and is Composer-managed + gitignored — **in-place edits
are overwritten on `composer install`.** To customize durably, create a subtheme under
`web/themes/custom/` (or override via your own recipe/config), don't edit byte_theme directly.

## AI stack

This install carries a large AI module suite, configured by `recipes/drupal_cms_ai`: `drupal/ai` core
plus `ai_agents`, `ai_dashboard`, `ai_image_alt_text`, `ai_assistant_api`, `ai_chatbot`, `canvas_ai`,
and providers `ai_provider_anthropic`, `ai_provider_openai`, `ai_provider_amazeeio` (amazee.ai is the
default and needs no key). Provider API keys are stored via the **Key** module + **easy_encryption**,
not in config — never hard-code them.

## What the repo tracks vs. what's generated

Per [.gitignore](.gitignore), everything Composer manages is **gitignored and must not be hand-edited
or committed**: `/vendor`, `/recipes`, `web/core`, `web/modules/contrib`, `web/themes/contrib`,
`web/profiles/contrib`, plus `web/sites/*/settings*.php`, `services*.yml`, `files/`, and `private/`.
Change contrib/core versions in [composer.json](composer.json), not on disk.

The repo's own source is: `composer.json`/`composer.lock`, [config/sync](config/sync), your code under
`web/modules/custom` and `web/themes/custom`, the local [web/themes/blank](web/themes/blank) theme, and
the docs/scaffold files.

## Guardrails (from AGENTS.md)

- Put custom code in `web/modules/custom` and `web/themes/custom`. Never edit Drupal core or contrib
  projects in place.
- Never commit secrets or machine-local overrides: `.env`, `settings.local.php`,
  `.ddev/config.local.yaml`, `auth.json`.
- Never commit `vendor/`, `recipes/`, contrib directories, or uploaded files under `web/sites/*/files`.

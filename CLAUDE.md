# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Horizon is Shopify's flagship first-party theme, built on Liquid Storefronts and theme blocks. This is a pure theme checkout — no `package.json`, build tooling, or test runner is present. Everything here is `.liquid`, `.json`, `.css`, and `.js` served directly to a Shopify store.

### Scope: this repo is for components and code, not merchandising

The connected theme (`edukid-storefront-horizon/main`) is edited from **two directions**: this repo pushes code via GitHub, and the **Shopify theme editor** pushes merchandising changes (colors, section/block settings, content) back to `main` as `shopify[bot]` commits. Both write to `config/settings_data.json`.

To avoid this repo's commits fighting the theme editor's auto-synced commits:

- **This repo/local dev is for**: creating and editing sections, blocks, snippets, CSS, JS, and schema *definitions* (`{% schema %}` settings, `config/settings_schema.json`).
- **Leave to the Shopify theme editor**: `config/settings_data.json` values, section/block instance settings, template composition (which sections/blocks are placed where with what settings) — anything that's really merchandising, not component code.
- Don't use `theme-skill` (below) to edit `config/settings_data.json` or template instance settings on this project unless explicitly asked — that's the theme editor's territory now.
- Before pushing, always fetch/check `origin/main` for new `shopify[bot]` sync commits rather than force-pushing (see Local development below).

**Bypass:** this is a default, not a hard rule. If the user explicitly asks to edit `config/settings_data.json` or compose template instance settings/merchandising (e.g. "update settings_data for X", "use theme-skill to build this page"), do it — just fetch/merge `origin/main` first so you're not clobbering a recent `shopify[bot]` sync commit.

## Commands

- Validate/lint: `shopify theme check` (via [Shopify CLI](https://shopify.dev/docs/storefronts/themes/tools/cli))
- Local dev server: `shopify theme dev --environment=development` (see Local development below — don't run bare `shopify theme dev`)
- CI runs Theme Check on every commit via `Shopify/theme-check-action`.

There are no other build/test commands in this checkout — schemas are edited directly in the `{% schema %}` block of each `.liquid` file (see Schemas below).

### Local development — always pin the dev theme

Connecting this repo to a store via the GitHub integration (Online Store → Themes) only syncs *code* into a draft theme on push; it does not give local `shopify theme dev` access to live store data. That still requires running the Shopify CLI dev server.

**Never run bare `shopify theme dev`.** With no `--theme`/`--environment` given, it seeds the new dev theme's settings from whatever theme is currently **published/live** on the store. If that live theme is a different theme (or an old/incompatible version of this one), the upload fails with a wall of schema errors (`Setting 'X' does not exist`, `default must be a color or dynamic source access path`, `Section type 'Y' does not refer to an existing section file`, etc.) — none of that means the theme code is broken, it means the seed data doesn't match this schema. It also litters the store's admin with a fresh `Development (...)` theme on every run.

`shopify.theme.toml` at the repo root pins the store and theme ID so dev sessions always target the same, already-compatible theme instead:

```toml
[environments.development]
store = "edukid-revamp.myshopify.com"
theme = "187660140824"  # edukid-storefront-horizon/main, the GitHub-connected draft theme
```

Run local dev with:

```sh
shopify theme dev --environment=development
```

Check `shopify theme list` if the target theme's ID ever changes (e.g. it gets deleted/recreated) and update `theme` in `shopify.theme.toml` to match.

## Architecture

### Directory roles

- `layout/` — `theme.liquid` (main wrapper) and `password.liquid`.
- `templates/` — JSON templates mapping a page type to an ordered list of sections (`{"sections": {...}, "order": [...]}`). Alternate templates follow `name.suffix.json`.
- `sections/` — Page-level Liquid components, each with its own `{% schema %}`. Files prefixed with `_` (e.g. `_blocks.liquid`) are private/shared partials, not directly selectable in the editor.
- `blocks/` — Theme blocks: reusable components nested inside sections or other blocks, each with its own `{% schema %}`. `_`-prefixed files follow the same private-partial convention.
- `snippets/` — `{% render %}`-only partials. No schema; must be documented with `{% doc %}`.
- `assets/` — Flat directory (no subfolders) of CSS/JS/images/icons, referenced via `asset_url`/`inline_asset_content`.
- `config/` — `settings_schema.json` (theme settings definitions) and `settings_data.json`.
- `locales/` — Translation JSON, `en.default.json` is the required source of truth; `en.default.schema.json` holds schema/editor-facing translation keys (e.g. the `names` section for `t:names.*` schema names).

### Sections vs. blocks vs. snippets

- **Sections** are top-level, editor-addable, always have a schema, and are referenced from templates or `{% content_for 'blocks' %}` areas.
- **Blocks** live inside sections/blocks, can be static (`{% content_for 'block', type: '...', id: '...' %}`, fixed placement, locked in editor) or dynamic (added via `{% content_for 'blocks' %}`). Only **one** `content_for 'blocks'` call is allowed per file — capture its output once if it's needed in multiple branches.
- **Snippets** have no schema and are purely `render`-based helper partials.
- Section/block IDs (`section.id`, `block.id`) are the only guaranteed-unique identifiers — always fold them into any DOM `id` you generate (CamelCase convention, e.g. `id="ProductCard-{{ block.id }}"`).

### The JS component framework

All custom elements should extend `Component` from `assets/component.js` (imported as `@theme/component`) rather than raw `HTMLElement`. It auto-wires `refs` (child elements marked with a `ref` attribute) and declarative `on:event="/methodName"` handler binding, and keeps `refs` in sync via a mutation observer. Zero external JS dependencies — everything is native browser APIs, no framework, no bundler-only syntax beyond what native ESM supports.

### Schemas

Schema settings/blocks/presets are written directly inside the `{% schema %}...{% endschema %}` tag of the section/block's `.liquid` file — there is no separate `schemas/` source directory or `build:schemas` step in this checkout. Schema `name` fields must use translation keys (`t:names.xxx`) resolved against `locales/en.default.schema.json`'s `names` section; add missing keys there.

### CSS scoping

Each section/block owns its styles in a `{% stylesheet %}...{% endstylesheet %}` tag scoped to its own class name (BEM). Shared/global styles belong in `assets/base.css`. Per-instance values are passed as inline CSS custom properties on the root element's `style` attribute (`--foo: {{ settings.foo }}`) rather than generating per-instance selectors like `.foo--{{ block.id }}`.

## Conventions worth knowing before editing

- **Liquid**: prefer inlining simple expressions directly in HTML attributes/output over pre-declaring `assign`/`capture` variables, unless the same complex value is reused or genuinely hard to read inline. Never invent Liquid tags/filters/objects. All snippets require `{% doc %}`/`{% enddoc %}` documentation with typed `@param`s.
- **CSS**: BEM naming, max `0 4 0` specificity, no IDs as selectors, no nesting beyond one level (except media queries and same-element state modifiers), logical properties for RTL support, mobile-first `min-width` media queries, never hardcode colors (use color scheme variables).
- **JS**: no external dependencies, `const` over `let`, `for...of` over `.forEach`, async/await over `.then()`, early returns over nested conditionals, events (`CustomEvent`, bubbling) for cross-component communication rather than direct coupling except parent→child method calls.
- **HTML**: prefer native elements over custom JS — `<details>/<summary>` for expand/collapse, `<dialog>` for modals, `popover` for tooltips/menus, `<search>` for search forms, native form validation attributes.
- **Localization**: all user-facing text goes through `{{ 'key' | t }}` and must be added to `locales/en.default.json`; only add English source strings, translators handle the rest.
- **Accessibility**: `<html lang>`, non-zoom-blocking viewport meta, no `title` attribute on non-iframe elements, mandatory descriptive `title` on iframes, a skip link targeting `tabindex="-1"` main content, `aria-hidden="true"` on all decorative SVG icons in `assets/`. Extensive additional per-component accessibility rules live in `.cursor/rules/*-accessibility.mdc` (accordion, carousel, modal, forms, tables, etc.) — check the relevant one before building/editing that component type.
- **Commits**: Conventional Commits format (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`), enforced by a local `commit-msg` hook and CI on both commits and PR titles.

## Rule source of truth

Detailed, topic-specific rules (one file per concern) live in `.cursor/rules/*.mdc` — consult the matching file before working in that area: `liquid.mdc`, `sections.mdc`, `blocks.mdc`, `snippets.mdc`, `templates.mdc`, `schemas.mdc`, `css-standards.mdc`, `javascript-standards.mdc`, `html-standards.mdc`, `theme-settings.mdc`, `locales.mdc`, `localization.mdc`, `assets.mdc`, plus one `*-accessibility.mdc` per interactive component pattern.

## Skills

- `theme-skill` (`.claude/skills/theme-skill/`) — JSON-only page-building: `templates/*.json`, section/block instance settings, `config/settings_data.json`. Given this project's scope (see "Scope" above), avoid this for `config/settings_data.json` or template instance settings unless explicitly asked — that's the Shopify theme editor's territory. `config/settings_schema.json` (setting *definitions*, not values) is still fair game as regular schema work. Never use it for creating or editing the sections/blocks/snippets themselves (new Liquid, JS, CSS) — that's regular component work governed by the conventions and `.cursor/rules/*.mdc` files above.

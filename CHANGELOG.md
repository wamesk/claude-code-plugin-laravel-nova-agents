# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-09-24

Aligns Nova building with Nova QA: `teamwork-task-test` 1.1.0 reviews every
task's diff for UI/UX, performance, security, and page reachability, and flags a
Nova resource that no menu item and no relation tab points at. Until now the
build side never asked for a click path, so such resources shipped findable only
by typing the URL.

The same release adds a fifth dimension, `framework`: write the idioms of the
Nova version the project actually has installed. The skill's examples target
Nova 5 while Nova 4 projects are still maintained, and code carried across the
major boundary fails outright — a Nova 5-typed override is a fatal error when the
class loads on Nova 4, a Nova 5-only method an undefined-method error on the
first request. `teamwork-task-test` 1.2.0 reviews `framework` as an advisory
dimension; here it is a build rule for new and changed code.

### Added

- `skills/wame-nova-patterns/reference/nova-cross-cutting-quality.md` — the four
  dimensions for Nova, plus `framework`, under the same keys QA uses, with code
  checked against the Nova 5.x source (and the Nova 4.x source where the majors
  differ):
  - `reachability`: `Nova::mainMenu()` with `MenuSection` / `MenuGroup` /
    `MenuItem::resource()`; `$displayInNavigation = false` only together with a
    relation field on the parent; inbound links (`BelongsTo` back-links,
    `Action::visit()->sole()`, detail-view links); renames that leave no stale
    URI keys. It states the traps: a custom `mainMenu()` hides every resource
    not listed in it, and `MenuItem::lens()` / `MenuItem::dashboard()` /
    `MenuSection::dashboard()` build fresh instances, so registration-time
    `canSee()` never reaches the menu while the route still enforces it
    (visible-but-403/404).
  - `security`: no registered policy means Nova allows everything; a policy
    without `viewAny` allows listing. A tenant scope placed only in
    `indexQuery()` leaves detail, edit, peek, preview, and `BelongsTo` dropdowns
    open. Actions gate the record with `canRun()` and report known failures via
    `Action::danger()` instead of throwing.
  - `performance`: `$with` / `withCount()` instead of per-row queries in fields.
    Nova's default search is `LIKE '%term%'`, which no B-tree index serves, so use
    `Column::exact()` (Nova 5 only — it does not exist in 4.29.5 or 4.35.11),
    `SearchableText` + a full-text index, or Scout (on ULID
    keys a plain `'id'` entry becomes `id LIKE '%…%'`). `->searchable()`
    `BelongsTo` on large tables; `$chunkCount` / queued actions.
  - `ui_ux`: translated labels and help with English keys that resolve. `canRun()`
    greys an action out without a reason, so hide it or explain via
    `Action::danger()`. Destructive actions extend `DestructiveAction` with a
    translated confirmation.
  - `framework`: detect `laravel/nova` and the Nova add-ons from `composer.lock`
    (zsh-safe `jq`); look APIs up for that major — Laravel Boost `search-docs`,
    context7 (`/websites/nova_laravel_v4` / `_v5`), nova.laravel.com, with
    `vendor/laravel/nova/src` as the final word; generate classes with
    `php artisan nova:*`, which prefers the project's published stubs. A Nova 4
    vs Nova 5 table checked against the 4.29.5 / 4.35.11 and 5.6.3–5.10.1
    sources and the v5 upgrade guide: platform, untyped vs typed `$query` in
    query-hook and `Filter::apply()` overrides, native `Tab` vs a tabs package,
    `Column::exact()` (Nova 5 only), enum `Select::options()`, Nova 5-only
    `->immutable()` / searchable filters / `$policy`, the removed `Place` field,
    the Vue / Inertia imports for custom components, and Heroicons v1 vs v2 menu
    icons. Nova idioms tagged with their minimum version (`Nova::mainMenu()`,
    `dependsOn()`, `->filterable()`, `Searchable*` 4.0; `Badge` / `Status`,
    `->copyable()`, `Repeater`, inline actions 4.x; `Tab`, `->immutable()`,
    enum options 5.0) and Nova guardrails (match the sibling resources, never
    carry code across the major boundary, leave untouched add-ons alone). The
    Laravel / PHP / Pest side points to `laravel-agents`.
  - A pre-finish self-check, including a `framework` line.
- `reference/nova-dusk-testing.md` → "Reachability — navigate via the menu":
  Dusk tests that start at `/nova` and click through `@sidebar-menu` or the
  parent's `@<child>-index-component` panel, plus a denied user who sees no
  entry and lands on `@403-error-page`. CRUD tests may still deep-link, but each
  new screen needs one click-through test.

### Changed

- `laravel-nova` agent: decision rules for all four dimensions, a DO NOT against
  unlinked screens and "hidden = protected", a Dusk test that navigates via the
  menu, and a pre-finish self-check before declaring done. It also reads the
  installed Nova major before writing, follows the `framework` rules (no Nova 5
  code in a Nova 4 project, built-in idioms over hand-rolled code, sibling
  patterns first, no drive-by rewrites), and gets `WebFetch`, Laravel Boost
  `application-info` / `search-docs`, and context7 in its `tools` so the docs
  lookup is callable from the subagent.
- `nova-resources.md` and `nova-actions-lenses-filters.md` checklists point to the
  new reference (reachability, policy + tenant scope, lens `query()` scoping,
  `canSee` / `canRun`, `DestructiveAction`).
- `wame-nova-patterns` SKILL.md lists the new reference and triggers on menu /
  reachability requests and on Nova-version questions; its "Supported versions"
  block now says the examples target Nova 5 and points Nova 4 projects to the
  `framework` table.

### Fixed

- `reference/nova-dusk-testing.md` → "Running the tests" told you to run the
  suite with `php artisan pest`, a command that does not exist (Pest registers
  only `pest:test`, `pest:dataset`, and `pest:dusk`), so the run stopped with
  `Command "pest" is not defined`. It now uses `./vendor/bin/pest`.
- `reference/nova-resources.md` → `BaseResource` declares
  `indexQuery(NovaRequest $request, Builder $query)` and the other query hooks
  with typed parameters — Nova 5's signatures — without saying so. Copied into a
  Nova 4 project, the class is a fatal error as soon as it loads:
  `Declaration of …::indexQuery(NovaRequest $request, Builder $query): Builder
  must be compatible with Laravel\Nova\Resource::indexQuery(NovaRequest $request,
  $query)` (reproduced against Nova 4.35.11; the same class loads on 5.10.1). A
  note under the example now gives the Nova 4 form — untyped `$query`, return
  types kept.

## [1.0.0] - 2026-07-24

### Added

- Subagent `laravel-nova` — a thin persona for building Laravel Nova admin panels
  (BaseResource, custom Fields/Actions/Lenses/Filters/Metrics, CSV/Excel export,
  Dusk browser tests) that defers to the `wame-nova-patterns` skill.
- Skill `wame-nova-patterns` — Nova resource/action/lens/filter/metric patterns and
  Laravel Dusk browser-testing reference, split across `reference/` for progressive
  disclosure.
- Designed to install alongside `laravel-agents`: the `pest-tester` agent picks up
  `wame-nova-patterns` for Nova/Dusk testing only when this plugin is present.

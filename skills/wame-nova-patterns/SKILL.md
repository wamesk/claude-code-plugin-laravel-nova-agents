---
name: wame-nova-patterns
description: "Use when building or reviewing a Laravel Nova admin panel — creating or editing Nova Resources (fields, panels/tabs, searchable columns, eager-loading queries), Actions, Lenses, Filters, Metrics or eCharts cards, translation keys, CSV/Excel export, sortable rows, or Dusk browser tests for Nova. Load it whenever the task mentions Nova resources, `App\\Nova\\*`, a BaseResource, Nova actions/lenses/filters/metrics, `outl1ne/nova-sortable`, or the `nova_notifications` table. Also holds the Nova cross-cutting quality rules (reachability via `Nova::mainMenu()` / relation tabs / inbound links, policies and tenant scope, eager loading and search indexes, translated help texts and destructive-action confirmation) and the framework rule (installed Nova version, Nova 4 vs 5 differences, Nova idioms by minimum version) — load it for 'add to the Nova menu', 'displayInNavigation', 'Nova policy', 'is this resource reachable', 'Nova 4 or Nova 5'."
---

# WAME Laravel Nova Patterns

Conventions and copy-ready code for building Laravel Nova admin panels: a shared
`BaseResource` abstract class, resource structure with panels, translation-key
patterns, Actions/Lenses/Filters/Metrics, eCharts cards, CSV/Excel export,
`nova-sortable` rows, and Dusk browser tests.

## When to use

- Creating or editing a Nova **Resource** (fields, panels/tabs, `title()`,
  `searchableColumns()`, `indexQuery()`/`detailQuery()` eager loading).
- Adding a Nova **Action** (simple or with fields, queued), **Lens**,
  **Filter** (select / boolean / date), or **Metric** (value / trend / partition).
- Building a custom **eCharts card** or wiring **CSV / Excel export**.
- Making rows drag-sortable with `outl1ne/nova-sortable`, or adding the
  `nova_notifications` table.
- Writing **Dusk** browser tests for Nova resources, actions, or filters.
- Adding, renaming, or removing any Nova **screen** (resource, lens, dashboard,
  tool) — make it reachable (menu entry or relation tab, plus inbound links),
  authorized (policy + tenant scope), fast (eager loading, index-friendly search),
  and understandable (translated help, confirmed destructive actions), and run the
  pre-finish self-check.
- Writing any Nova code in a project whose Nova major you have not checked —
  read `laravel/nova` from `composer.lock` first; Nova 4 and Nova 5 code are not
  interchangeable (`framework` in `reference/nova-cross-cutting-quality.md`).

## Naming note

Code examples use `Vendor` as a placeholder for the vendor PHP namespace root and
a module segment such as `User` or `Post` (e.g. `Vendor\User\Nova\User`). The real
namespace root, the on-disk package directory, and the Composer package name are
each defined per-project in `CLAUDE.md` and may differ from one another.

## Supported versions

- **Laravel Nova** 5.x
- **Laravel** the project's supported Laravel version
- **PHP** 8.4+
- **Pest** the project's supported Pest version (browser tests run through Laravel Dusk)
- Optional packages: `outl1ne/nova-sortable` (drag ordering),
  `maatwebsite/excel` (XLSX export)

The examples target Nova 5. Nova 4 projects (4.29–4.35) are still maintained, and
some of the Nova 5 code here fails on them — typed `Builder $query` overrides,
`Column::exact()`, native `Tab`. The installed version decides: read it from
`composer.lock` and apply the Nova 4 column of the table in
[reference/nova-cross-cutting-quality.md](reference/nova-cross-cutting-quality.md)
→ `framework`.

## Reference

- [reference/nova-resources.md](reference/nova-resources.md) — `BaseResource`
  abstract class, basic + advanced (panels) resource structure, module layout,
  translation-key pattern and translation file example.
- [reference/nova-actions-lenses-filters.md](reference/nova-actions-lenses-filters.md)
  — Actions (simple, with fields, queued), Lenses, Filters (select/boolean/date),
  Metrics (value/trend/partition), custom eCharts card, CSV & Excel export.
- [reference/nova-dusk-testing.md](reference/nova-dusk-testing.md) — Dusk browser
  tests for resources / actions / filters, reachability tests that navigate via the
  sidebar menu and relation panels, a reusable Nova test-helper trait, and the
  commands to run them.
- [reference/nova-cross-cutting-quality.md](reference/nova-cross-cutting-quality.md)
  — the five build-time quality dimensions with the same keys QA uses
  (`reachability`, `security`, `performance`, `ui_ux`, `framework`):
  `Nova::mainMenu()` with `MenuSection` / `MenuGroup` / `MenuItem`,
  `$displayInNavigation` + relation tabs, inbound links, policies and tenant
  scope across every query hook, `$with` and index-friendly search,
  disabled/destructive actions, Nova version detection with the Nova 4 vs Nova 5
  differences and the built-in Nova idioms by minimum version, and a pre-finish
  self-check.
- [reference/nova-database.md](reference/nova-database.md) — drag-sortable rows
  with `HasSortableRows`, the `sort` column convention, and the
  `nova_notifications` table migration.

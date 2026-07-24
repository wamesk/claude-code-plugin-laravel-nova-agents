---
name: wame-nova-patterns
description: "Use when building or reviewing a Laravel Nova admin panel — creating or editing Nova Resources (fields, panels/tabs, searchable columns, eager-loading queries), Actions, Lenses, Filters, Metrics or eCharts cards, translation keys, CSV/Excel export, sortable rows, or Dusk browser tests for Nova. Load it whenever the task mentions Nova resources, `App\\Nova\\*`, a BaseResource, Nova actions/lenses/filters/metrics, `outl1ne/nova-sortable`, or the `nova_notifications` table."
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

## Reference

- [reference/nova-resources.md](reference/nova-resources.md) — `BaseResource`
  abstract class, basic + advanced (panels) resource structure, module layout,
  translation-key pattern and translation file example.
- [reference/nova-actions-lenses-filters.md](reference/nova-actions-lenses-filters.md)
  — Actions (simple, with fields, queued), Lenses, Filters (select/boolean/date),
  Metrics (value/trend/partition), custom eCharts card, CSV & Excel export.
- [reference/nova-dusk-testing.md](reference/nova-dusk-testing.md) — Dusk browser
  tests for resources / actions / filters, a reusable Nova test-helper trait, and
  the commands to run them.
- [reference/nova-database.md](reference/nova-database.md) — drag-sortable rows
  with `HasSortableRows`, the `sort` column convention, and the
  `nova_notifications` table migration.

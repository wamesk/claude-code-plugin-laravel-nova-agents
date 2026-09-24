# laravel-nova-agents

The Laravel Nova subagent and companion skill for Claude Code, by [WAME](https://wame.sk).

Part of the `wame` marketplace. Install **alongside** [`laravel-agents`](https://github.com/wamesk/claude-code-plugin-laravel-agents) — this plugin adds the Nova-specific layer that `laravel-agents` deliberately leaves out.

## Installation

```
/plugin marketplace add wamesk/claude-code
/plugin install laravel-agents@wame
/plugin install laravel-nova-agents@wame
```

## Agent

| Agent | Use it for |
|-------|-----------|
| **laravel-nova** | Building Laravel Nova admin panels — the `BaseResource` abstract class, custom Fields, Actions, Lenses, Filters, Metrics, CSV/Excel export, and Laravel Dusk browser tests. Uses `title()`/`searchableColumns()`, eager loading, help text, and tabs, makes every new screen reachable from the menu or a relation tab, and writes for the Nova major the project has installed (Nova 4 and Nova 5 code are not interchangeable). |

## Skill

| Skill | What it holds |
|-------|---------------|
| **wame-nova-patterns** | Nova resource templates, custom Actions/Lenses/Filters/Metrics, CSV/Excel export, Laravel Dusk browser-testing patterns, and the Nova cross-cutting quality reference (reachability, security, performance, UI/UX, framework — Nova 4 vs Nova 5). Split across `reference/` for progressive disclosure. |

## How it composes with `laravel-agents`

The `pest-tester` agent (from `laravel-agents`) invokes `wame-nova-patterns` **only if it is
available**. Install this plugin in a Nova project and `pest-tester` will write Nova/Dusk browser
tests; leave it out in a non-Nova project and `pest-tester` skips Nova testing entirely. That
keeps the universal Laravel agents free of Nova assumptions.

## Cross-cutting quality (since 1.1.0)

[`teamwork-task-test`](https://github.com/wamesk/claude-code-plugin-teamwork-task-test) reviews every
task's diff for UI/UX, performance, security, and page reachability, plus an advisory fifth dimension,
`framework`. `laravel-nova` applies all five **while building**, under the same keys — see
`skills/wame-nova-patterns/reference/nova-cross-cutting-quality.md`:

| Key | Nova build-time rule (short) |
|-----|------------------------------|
| `reachability` | A new Resource / Lens / Dashboard / Tool is in `Nova::mainMenu()` (`MenuSection` / `MenuGroup` / `MenuItem::resource()` …) with visibility matching the policy, or — for a child — `$displayInNavigation = false` **only** with a `HasMany` / `BelongsToMany` / `MorphMany` field on the parent. A project with a custom `mainMenu()` hides every resource not listed there, even when it is registered. Inbound links: `BelongsTo` back-links, `Action::visit()`, detail-view links. |
| `security` | A registered policy with `viewAny` and object-scoped `view` / `update` / `delete`; tenant scope across `indexQuery`, `detailQuery`, `editQuery`, `relatableQuery`, `scoutQuery`, and lenses (or a global scope); `canRun()` per record. |
| `performance` | `$with` / `indexQuery()` eager loading, `withCount()` instead of per-row queries in fields, index-friendly `searchableColumns()` (exact / full-text / Scout), `->searchable()` `BelongsTo` on large tables. |
| `ui_ux` | Labels and help texts through `__()` with English keys in the module lang file; unavailable actions hidden or explained; destructive actions extend `DestructiveAction` with a translated confirmation. |
| `framework` | Read `laravel/nova` and the Nova add-ons from `composer.lock` first; look APIs up for that major (Laravel Boost `search-docs` → context7 → nova.laravel.com/docs/v4 or v5; the vendor source is the final word); generate classes with `php artisan nova:*`. The examples target Nova 5 — on Nova 4 keep `$query` untyped in query-hook and filter overrides (a typed `Builder $query` is a fatal on load) and skip `Column::exact()`, native `Tab`, `->immutable()`, and enum `Select::options()`. Prefer built-in idioms tagged with their minimum Nova version (`dependsOn()`, `->filterable()`, `Badge` / `Status`, `->copyable()`, `Repeater`, inline actions, native tabs on 5) over hand-rolled code or a new add-on, but match the sibling resources and never rewrite untouched code. |

Dusk tests for a new screen navigate via the sidebar menu or the parent's relation panel, never by
URL — see `reference/nova-dusk-testing.md`. The non-Nova counterpart lives in `laravel-agents`
(`wame-laravel-standards` → `reference/cross-cutting-quality.md`).

## License

MIT

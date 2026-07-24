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
| **laravel-nova** | Building Laravel Nova admin panels — the `BaseResource` abstract class, custom Fields, Actions, Lenses, Filters, Metrics, CSV/Excel export, and Laravel Dusk browser tests. Uses `title()`/`searchableColumns()`, eager loading, help text, and tabs. |

## Skill

| Skill | What it holds |
|-------|---------------|
| **wame-nova-patterns** | Nova resource templates, custom Actions/Lenses/Filters/Metrics, CSV/Excel export, and Laravel Dusk browser-testing patterns. Split across `reference/` for progressive disclosure. |

## How it composes with `laravel-agents`

The `pest-tester` agent (from `laravel-agents`) invokes `wame-nova-patterns` **only if it is
available**. Install this plugin in a Nova project and `pest-tester` will write Nova/Dusk browser
tests; leave it out in a non-Nova project and `pest-tester` skips Nova testing entirely. That
keeps the universal Laravel agents free of Nova assumptions.

## License

MIT

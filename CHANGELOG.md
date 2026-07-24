# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

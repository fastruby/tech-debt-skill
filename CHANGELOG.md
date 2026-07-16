# Changelog

All notable changes to this skill are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Single self-contained HTML report (`index.html`) as the primary deliverable, with an
  executive summary and 100-point health score at the top, one section per tool, embedded
  RubyCritic screenshots, and the top 3 recommended actions at the end.
- Timestamped output directory (`tech-debt-audit-YYYYMMDD-HHMMSS/`) holding the report plus
  `raw/`, `rubycritic/`, and `screenshots/`, so each run is a baseline you can keep.
- Trivy filesystem scan (vulnerable dependencies, leaked secrets, and misconfigurations).
- Exception tracking check (Sentry, Honeybadger, Rollbar, Bugsnag, Airbrake, etc.) and
  performance monitoring check (New Relic, Scout, Skylight, AppSignal, rack-mini-profiler);
  the absence of either is flagged as a departure from best practices.
- `rails_stats` for codebase statistics and `bundle-stats` for dependency weight (transitive
  dependencies per gem).
- Report styled with the [FastRuby.io styleguide](https://fastruby.github.io/styleguide)
  palette and Oxygen font; mobile and tablet friendly (tables scroll on small screens).
- A link to the open source project for every tool, both in its section and in the appendix.
- Color-coded health score: red under 50, yellow 50-74, green 75-100.
- When Ruby or Rails is out of date, the upgrade recommendation offers a DIY path (the
  [claude-code_rails-upgrade-skill](https://github.com/ombulabs/claude-code_rails-upgrade-skill))
  and a help path ([FastRuby.io](https://www.fastruby.io)).

### Changed
- Replaced Rails' native `rake stats` with `rails_stats`.

### Removed
- All references to Code Climate (a discontinued service).

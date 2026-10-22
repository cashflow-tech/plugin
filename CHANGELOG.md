# Changelog

## 0.4.1 (2026-10-22)

- trips: load existing trips from one paged `tag_report` call instead of one query per tag; cap cashflow calls at 3 per message; bound per-trip detail to the trip's dates; report every trip in one call

## 0.4.0 (2026-04-14)

### New Skills
- **optimize-subs** — Subscription audit with Apple receipt email parsing
- **optimize-cards** — Credit card rewards optimization
- **burn** — Burn rate and runway analysis
- **fishy** — Anomaly detection and suspicious charge audit
- **setup** — First-time onboarding guide

### Improvements
- Improved skill descriptions for better triggering
- Added "why" sections to skills explaining when they're useful

## 0.3.5 (2026-04-13)

- Improved tidy skill with incremental cleanup workflow
- Added clusters query mode support to tidy
- Added apply_rules steps and re-run guidance
- Date filter examples in tidy skill

## 0.3.4 (2026-04-12)

- Filter Phase 1 to expenses only in tidy skill
- Rewrite tidy skill for incremental cleanup

## 0.3.2 (2026-04-10)

- Added tag-trips skill for travel expense organization
- Improved recap skill with year-ago seasonal comparison

## 0.3.1 (2026-04-09)

- Initial public release with recap and tidy skills
- OAuth authentication with cashflow.tech
- Three MCP tools: query, annotate, admin

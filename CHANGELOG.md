## [0.2.0](https://github.com/precepts-dev/standards/compare/v0.1.4...v0.2.0) (2026-04-12)

### Features

* **intg:** populate INTG-GOV-001 integration styles governance ([#2](https://github.com/precepts-dev/standards/issues/2)) ([69e928b](https://github.com/precepts-dev/standards/commit/69e928bdcffbba6d48e540b699f9bd0f67b8cd3e))

## [0.1.4](https://github.com/precepts-dev/standards/compare/v0.1.3...v0.1.4) (2026-04-12)

### Bug Fixes

* **ci:** add semantic-release, GitHub App token, and dev workflow ([#1](https://github.com/precepts-dev/standards/issues/1)) ([0d8edd3](https://github.com/precepts-dev/standards/commit/0d8edd3a86db83c67a675a90938b26ea4bb508cc))

# Changelog

All notable changes to `@precepts/standards` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This file is maintained automatically by [semantic-release](https://github.com/semantic-release/semantic-release) — do not edit manually.

<!-- semantic-release inserts new entries above this line -->

## [0.1.3] - 2026-04-12

### Features

- **INTG-STD-003**: Filled money and currency standard — canonical `{"amount": "string", "currency_code": "ISO-4217"}` representation, precision rules, rounding (HALF_EVEN), exchange rate structure, validation regex

### Bug Fixes

- **INTG-STD-004**: Fixed approved-abbreviations table header row (was rendered as data, not header)
- **INTG-STD-004**: Converted multi-statement rules to bullet-point format throughout
- **INTG-STD-004**: Added inline rationale for R-7 (`is_` prefix), R-14 (path param casing), R-15 (no `/api`), R-16 (3-level nesting), R-18–R-22
- **INTG-STD-006**: Renumbered sub-rules from `R-COMPAT-NNN` to `R-N-M` format for hierarchical clarity; expanded R-2-3 (unknown enum handling)
- **INTG-STD-008**: Clarified snake_case path parameter rationale; split R-2 into individual bullets; added `total_count` opt-in rationale
- **INTG-STD-015**: Clarified CloudEvents `type` naming vs broker topic naming; documented claim-check pattern; added Kafka partition key ordering rationale
- **INTG-STD-029**: Added W3C Trace Context term definitions; defined ULID; explained OpenTelemetry curly-brace unit notation
- **INTG-STD-033**: Added bulkhead pattern explanation; corrected incomplete standard IDs in R-6 composition flow; clarified R-7 scope vs INTG-STD-029
- **INTG-STD-034**: Defined MITM (Man-in-the-Middle) attack in R-9
- **INTG-STD-035**: Justified deadline budget enforcement in R-5; explained Slowloris and Slow POST attacks in R-9

## [0.1.2] - 2026-03-28

### Bug Fixes

- Switched publish job to Node.js 24 to resolve Trusted Publishing 404 error

## [0.1.1] - 2026-03-28

### Features

- Added `repository_dispatch` to notify `precepts-dev/platform` repo after every successful publish

## [0.1.0] - 2026-03-28

### Features

- Initial release of `@precepts/standards` with integration standards suite

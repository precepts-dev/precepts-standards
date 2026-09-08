## [0.5.0](https://github.com/precepts-dev/standards/compare/v0.4.3...v0.5.0) (2026-09-08)

### Features

* **integration:** add integration standards for API design ([#24](https://github.com/precepts-dev/standards/issues/24)) ([926c916](https://github.com/precepts-dev/standards/commit/926c91627eda32d924411662eaa3ec7999291adc))

## [0.4.3](https://github.com/precepts-dev/standards/compare/v0.4.2...v0.4.3) (2026-08-24)

### Bug Fixes

* **directory:** Rename product requirement document path ([#23](https://github.com/precepts-dev/standards/issues/23)) ([4c51829](https://github.com/precepts-dev/standards/commit/4c518290c2949e772c081effcd90a512952a6e4b))

## [0.4.2](https://github.com/precepts-dev/standards/compare/v0.4.1...v0.4.2) (2026-07-06)

### Bug Fixes

* **ci:** remove special character ([#21](https://github.com/precepts-dev/standards/issues/21)) ([1734d51](https://github.com/precepts-dev/standards/commit/1734d5114f9fb7e87c38f4792ee5d8ba43252e00))

## [0.4.1](https://github.com/precepts-dev/standards/compare/v0.4.0...v0.4.1) (2026-06-16)

### Bug Fixes

* **ux:** Add initial version entry for accessibility standards ([#16](https://github.com/precepts-dev/standards/issues/16)) ([fc6c882](https://github.com/precepts-dev/standards/commit/fc6c88260be791e188d02c5c74b46eb121bea6ba))

## [0.4.0](https://github.com/precepts-dev/standards/compare/v0.3.1...v0.4.0) (2026-06-16)

### Features

* **ux:** Add UX accessibility, interaction, color tokens ([#15](https://github.com/precepts-dev/standards/issues/15)) ([8107fa6](https://github.com/precepts-dev/standards/commit/8107fa6aa10ea9bc07c9355109e6f2a6adefcccf))

## [0.3.1](https://github.com/precepts-dev/standards/compare/v0.3.0...v0.3.1) (2026-05-05)

### Bug Fixes

* **security-logging:** MDX render issue for platform ([#11](https://github.com/precepts-dev/standards/issues/11)) ([14ad763](https://github.com/precepts-dev/standards/commit/14ad763924a8358c283c552b5b3daf51681e69ec))

## [0.3.0](https://github.com/precepts-dev/standards/compare/v0.2.4...v0.3.0) (2026-05-05)

### Features

* **sec:** add complete security domain standards (18 documents) ([#10](https://github.com/precepts-dev/standards/issues/10)) ([ac6411f](https://github.com/precepts-dev/standards/commit/ac6411f767a1b56618c7576e75bcd0de6ee80fb2))

## [0.2.4](https://github.com/precepts-dev/standards/compare/v0.2.3...v0.2.4) (2026-04-13)

### Bug Fixes

* **INTG-STD-003:** correct broken relative link to naming-conventions ([#6](https://github.com/precepts-dev/standards/issues/6)) ([b007272](https://github.com/precepts-dev/standards/commit/b007272b2d6f9814a969803c18317db6bc4ca5f4))

## [0.2.3](https://github.com/precepts-dev/standards/compare/v0.2.2...v0.2.3) (2026-04-13)

### Bug Fixes

* **INTG-GOV-001:** Dummy trigger for standards update on platform ([#5](https://github.com/precepts-dev/standards/issues/5)) ([1224b20](https://github.com/precepts-dev/standards/commit/1224b2051e18da5b5076080001272e93edaa01c0))

## [0.2.2](https://github.com/precepts-dev/standards/compare/v0.2.1...v0.2.2) (2026-04-13)

### Bug Fixes

* **INTG-GOV-001:** Version details section ([#4](https://github.com/precepts-dev/standards/issues/4)) ([8c28107](https://github.com/precepts-dev/standards/commit/8c281074b7cbf70df3ed5881d2f85741b7b518b1))

## [0.2.1](https://github.com/precepts-dev/standards/compare/v0.2.0...v0.2.1) (2026-04-12)

### Bug Fixes

* **intg-gov-001:** Update version history description from 'Initial s… ([#3](https://github.com/precepts-dev/standards/issues/3)) ([9e418f4](https://github.com/precepts-dev/standards/commit/9e418f4d805558cbbbdae08fe03b3368ab521bdc))

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

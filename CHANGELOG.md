# Changelog

All notable changes to **Aqly Software Team** are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed

- **Licence — added a free-use grant.** Personal use, internal business use, modification for your own
  use, and producing work product for yourself, your business, or your clients are now permitted at
  no charge. Redistribution, resale, sublicensing, repackaging, and building a competing product for
  distribution remain prohibited, and now apply to free users and purchasers alike. Work product you
  create with the skill is yours and is not restricted. The grant may be withdrawn for future
  versions but cannot be revoked for a version already obtained under it.
- **Install instructions** now point at the skill folder (`.../tree/main/software-team`) rather than the
  repository root, which contains no `SKILL.md` and cannot be imported.

> These changes are live on `main`, so they already reach anyone importing from it, but they are not
> yet covered by a version tag.

## [1.0.0] — 2026-08-01

### Added

- Initial public release.
- 40 agents spanning product, design, architecture, implementation, verification, operations, and the council layer.
- Three modes: **BUILD** (greenfield), **IMPROVE** (change an existing repo in place on an isolation git branch), and **COUNCIL** (self-improvement operations).
- Six engineering standards: coding standards, definition of done, stage output contracts with 17 profile flags, tech-stack policy, failure-recovery routing, and council-complexity thresholds.
- Append-only run and council log schemas. Logs are created on first run, so the package ships clean.
- Gate discipline: a `verdict: pass` without pasted command output is treated as a failure; failures are classified before routing; retry cap of 3 per gate, then an honest `BLOCKED.md` stop.

[Unreleased]: https://github.com/tohir-dev/aqly-software-team/compare/v1.0.0...main
[1.0.0]: https://github.com/tohir-dev/aqly-software-team/releases/tag/v1.0.0

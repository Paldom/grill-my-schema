# Changelog

All notable changes to this repository's skills are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning: [SemVer](https://semver.org) on the plugin manifest
(breaking skill-interface change → major, new skill → minor, fix → patch).

## [Unreleased]

### Added
- `grill-my-schema` skill: interrogates the design of an application-serving
  schema built on an analytics gold layer — prioritized challenging questions
  (ten fatal-flaw questions + twelve axes), decision log, gap list, and risk
  register, before DDL exists. Platform-independent; primary-source-verified
  references.
- `serving-schema-review` skill: reviews a concrete app-serving schema (DDL,
  ERD, table definitions, or data contract) against a ten-section rubric with
  severity calibration, separate evidence-gap reporting, and untrusted-artifact
  handling.
- `docs/setup-prompt.md`: paste-ready session prompt composing the two skills
  into a grill → design → review loop.
- Repository scaffolded from the skills template.

# Changelog

All notable changes to this repository's skills are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning: [SemVer](https://semver.org) on the plugin manifest
(breaking skill-interface change → major, new skill → minor, fix → patch).

## [Unreleased]

## [0.2.0] - 2026-08-31

### Added
- Adopted the current skillskit gate: executed trigger evals scoring every trigger
  prompt against every skill description (rank-1 routing accuracy 95.0%), a security
  scan over skill content and bundled scripts, ruff lint and format, README-shape
  validation, pre-commit hooks and a write-time lint hook.

### Changed
- Skill descriptions sharpened where the eval gate showed a sibling outranking a
  skill on its own trigger prompts, or a stated non-trigger matching better than any
  trigger. Fixes changed the scope boundary, not just the wording.

### Fixed
- Findings the new lint gate surfaced in this repo's own scripts, fixed at the
  source; where a rule was wrong for a line it is suppressed there with its reason.


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

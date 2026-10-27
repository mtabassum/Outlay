# Release Notes

> **Format contract** — parsed by `GET /api/v1/release-notes`. Keep headings exactly: `## [x.y.z] - YYYY-MM-DD` (or `## [Unreleased]`), then `### Added|Changed|Fixed|Security|Known issues|QA: what to verify`. One bullet = one line. The newest release is first.

## [Unreleased]

### Added
- (empty — implementation not started; add bullets here as slices merge)

### QA: what to verify
- (nothing yet)

## [0.0.1] - 2026-10-28

### Added
- Documentation baseline: PRD, AppFlow, UiSpec, TRD, Schema, Plan, Master Specification, Master Prompt, Roadmap, AI Handover, AI brain (CLAUDE.md).
- Architecture decision records (ADR-0001…0013) and C4/ER diagrams.
- Backend and Frontend developer guides; README with usage guide.

### Known issues
- The source repository could not be read during authoring; its contents are unverified (A-11).
- Free SMS gateway choice (A-04) and MS Access limitations (A-06) need confirmation.

### QA: what to verify
- Read `MASTER-SPECIFICATION.md §9–10` and confirm or change the decisions/open questions.
- Confirm every item of the original requirement PDF maps to a requirement ID (MVP list: sign-up, invitation, login with org context, categories, expenses, yearly budget Soft, 80% alert, Monthly Summary, Super Admin org list).

<!-- Template for future releases
## [0.1.0] - YYYY-MM-DD
### Added
- …
### Changed
- …
### Fixed
- …
### Security
- …
### QA: what to verify
- Steps QA can follow with expected results (include role + credentials from README).
-->

# Outlay — Release Notes

This file is the human-readable source for the **Release Notes API** (`GET /api/v1/system/release-notes`).
Every release has a matching entry in `src/Backend/Outlay.Api/Resources/release-notes.json`, and CI fails if the versions differ.
SQA: use the **What to test** checklist of each version. Run it with the listed roles (Postman folders `01`–`05`).

Format: [Keep a Changelog](https://keepachangelog.com/) + [Semantic Versioning](https://semver.org/). Each version has *Highlights*, *Added*, *Changed*, *Fixed*, *Security*, *Breaking changes*, *Known issues*, and *What to test (SQA)*.

---

## [Unreleased]

### Added
- _Nothing yet. Implementation starts with milestone M0 (see docs/Plan.md)._

### What to test (SQA)
- _Each task appends checklist lines here (area, steps, expected result, roles)._

---

## [0.0.0] — 2026-10-28 — Specification baseline

### Highlights
- Complete specification set for Outlay, a multi-tenant expense tracking and budget control SaaS. No application code yet.

### Added
- **Core specs:** PRD, AppFlow, UiSpec, TRD, Schema and Plan (milestones M0–M13 with a Definition of Done).
- **Architecture:** 18 ADRs and C4 diagrams (context, containers, API and web components, API by project, expense-approval and log-pipeline dynamic views, deployment, code level).
- **Guides:** Backend and Frontend Developer Guides with worked "build a CRUD feature" examples.
- **Governance:** CLAUDE.md, MASTER-SPECIFICATION (traceability matrix, open questions, limitations and deviations), MASTER-PROMPT, ROADMAP and AI-HANDOVER.
- **Key decisions:**
  - Clean Architecture + vertical slices + CQRS with MediatR 14 (Community licence) and OpenIddict + ASP.NET Core Identity with BCrypt.
  - Delegated permissions, shared-database multi-tenancy, and seven database providers in support tiers.
  - FusionCache, categorised Serilog logs and an offline-first PWA.
  - A deterministic assistant and manual mapping (no AutoMapper).
  - Default test stack: xUnit, NSubstitute, AwesomeAssertions, Testcontainers, BenchmarkDotNet, k6.
- **Business rules confirmed:**
  - Budgets are monthly allowances.
  - Expenses can be edited only while Saved/Pending in an open month, and every edit re-runs all checks.
  - Over-budget amounts show as "Over by ৳…".
  - Managers see the whole organisation by default, and approvals follow each Manager's scope.
- **Repo alignment:** specs match the actual repository layout (`Outlay.slnx` at root, `src/Backend`, `src/Frontend/outlay-web`). Ports are documented with their defaults and how to override them.

### Known issues
- Open questions with working assumptions: Q-05 (production SMS provider), Q-07…Q-12, Q-14 (MASTER-SPECIFICATION §6).
- Limitations and deliberate deviations from the brief are listed in MASTER-SPECIFICATION §7.

### What to test (SQA)
| # | Area | Steps | Expected result | Roles |
|---|---|---|---|---|
| 1 | Documentation | Open MASTER-SPECIFICATION.md and follow every link | All documents and ADRs open | — |
| 2 | Diagrams | Open `docs/diagrams/C4-DIAGRAMS.md`, `docs/AppFlow.md` and `docs/Schema.md` on GitHub | Every Mermaid diagram renders, and no raw Mermaid text is visible | — |
| 3 | Traceability | Pick 5 items from MASTER-SPECIFICATION §4 and open their referenced sections | Each requirement is specified where stated | — |

---

<!--
TEMPLATE — copy for each release

## [x.y.z] — YYYY-MM-DD — <short title>

### Highlights
- …

### Added
- <Feature> (FR-xxx, task Mx-yy)

### Changed
- …

### Fixed
- <Bug> (correlation/issue id)

### Security
- …

### Breaking changes
- …

### Known issues
- …

### What to test (SQA)
| # | Area | Steps | Expected result | Roles |
|---|---|---|---|---|
| 1 | Expenses | Log in as Member → Add ৳1,200 in Travel (Hard limit, over budget) | Button reads "Submit for approval"; status Pending; approvers (Admins + Managers whose scope covers the member) notified | Member, Manager |
-->

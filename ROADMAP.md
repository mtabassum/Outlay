# ROADMAP

Status legend: ⬜ not started · 🟦 in progress · ✅ done · Dates are targets, not promises. Current status: **documentation baseline only (0.0.1)**; no application code has been verified in the repo yet (A-11).

## Release train

| Release | Theme | Scope (milestones) | Exit criteria | Status |
|---|---|---|---|---|
| **0.0.1** | Documentation baseline | Specs, ADRs, C4, guides | Docs reviewed & approved | ✅ |
| **0.1.0** | **Foundation MVP** (matches original MVP list) | M0–M5 | Sign-up → onboarding → login → categories → budgets (Soft) → expenses (own only) → 80% in-app alert → Monthly Summary → Super Admin org list; InMemory DB; 5 log categories; EN/BN; Scalar; release-notes API; isolation tests green | ⬜ |
| **0.2.0** | **Governance & Security** | M6–M9 | Hard limit + approvals, adjustments, lockout, password reset/change, MFA (dev OTP 111111), SMS/email, permissions page, advanced search, bulk import with full error report | ⬜ |
| **0.3.0** | **Scale & Portability** | M10–M11 | All DB providers + Dapper/EF switch, backup API, Redis/HybridCache, resilience, rate limiting, aggregates | ⬜ |
| **0.4.0** | **Offline & Intelligence** | M12–M13 | Installable PWA, offline CRUD + sync, assistant (text+voice) with guardrail tests | ⬜ |
| **1.0.0** | **Production** | M14–M16 | Load/stress/perf suites with baselines, CI/CD, Docker+LB, full docs, regression green | ⬜ |
| 1.1+ | Enterprise | SSO (OIDC/SAML), custom reports, data export/archive, billing integration, SignalR live updates | – | ⬜ |
| Later | Ideas | Receipt OCR, bank-feed import, AI duplicate detection (US-21), mobile wrappers, Kubernetes/Helm | – | ⬜ |

## Milestone tracker
| M | Name | Rel | Status |
|---|---|---|---|
| M0 | Repo & scaffolding | 0.1 | ⬜ |
| M1 | Core platform (logging, config, errors, InMemory) | 0.1 | ⬜ |
| M2 | Identity v1 | 0.1 | ⬜ |
| M3 | Tenancy & onboarding | 0.1 | ⬜ |
| M4 | Categories, budgets, expenses | 0.1 | ⬜ |
| M5 | Reports, dashboard, FE MVP | 0.1 | ⬜ |
| M6 | Governance (hard limit, approvals, adjustments) | 0.2 | ⬜ |
| M7 | Security hardening (lockout, MFA, SMS) | 0.2 | ⬜ |
| M8 | Permission delegation | 0.2 | ⬜ |
| M9 | Advanced search & bulk import | 0.2 | ⬜ |
| M10 | Providers, Dapper, backup | 0.3 | ⬜ |
| M11 | Cache, resilience, rate limit | 0.3 | ⬜ |
| M12 | PWA & offline | 0.4 | ⬜ |
| M13 | Assistant | 0.4 | ⬜ |
| M14 | Quality gates | 1.0 | ⬜ |
| M15 | DevOps & docs | 1.0 | ⬜ |
| M16 | 1.0 release | 1.0 | ⬜ |

## Scope-control note
The brief is intentionally large (7 data stores, 2 ORMs, offline, voice, 5 test suites). Slicing keeps every release shippable. If time is short, defer in this order: Access provider → Memcached → Oracle → Enterprise SSO → voice (text assistant stays).

## Upgrade policy
* .NET: adopt each GA release within one quarter; LTS preferred for production; ADR + release note per bump.
* Next.js/React: stay on latest stable; weekly canary CI against `@latest`.
* Dependencies: Renovate weekly; security patches within 72 h.

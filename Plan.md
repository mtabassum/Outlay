# Plan — Build Order, Milestones, Definition of Done

Rules: **vertical slices, small commits, green tests before moving on, no regressions.** One milestone ≈ one PR series. Each milestone ends with: tests green · docs touched (`RELEASENOTES.md`, `AI-HANDOVER.md`) · Scalar/Postman updated · EN+BN strings · logs verified.

## Definition of Done (every slice)
- [ ] Command/Query + handler + validator + endpoint in one feature folder
- [ ] Permission attribute + tenant filter + audit entry (for writes)
- [ ] Unit tests + integration test (incl. wrong-tenant & wrong-role cases)
- [ ] OpenAPI title/description/examples; Postman request with test script
- [ ] EN + BN message keys; no hard-coded text; no sensitive data in logs
- [ ] Advanced search + bulk import wired if the slice is a CRUD (use the generic kits)
- [ ] FE screen (if any): loading/empty/error/offline/forbidden, both languages, dark/light, mobile

## Milestones

| # | Milestone | Key deliverables | Req IDs | Rel |
|---|---|---|---|---|
| M0 | **Repo & scaffolding** | Solution/projects, `Directory.Build.props`, `global.json`, central packages, `.editorconfig`, analyzers, `web/` Next app, git hooks, CI skeleton, Dockerfiles skeleton, `.env.example`, architecture tests | FR-DEV-01, NFR-04/05 | R1 |
| M1 | **Core platform** | Config system (appsettings + env), `OutlayLogging` (5 categories, redaction, advisor, query interceptor), correlation, idempotency, ProblemDetails, localization infra, health, InMemory persistence + generic repository/UoW, Scalar + theme, release-notes API, log-download API | FR-LOG-01, FR-CORR-01, FR-IDEM-01, FR-DEG-01, FR-DB-01(InMemory), FR-OPS-02/03/04, FR-I18N-01, FR-DOC-01 | R1 |
| M2 | **Identity v1** | OpenIddict, BCrypt, login-ticket grant, refresh rotation/revocation, RBAC permission policies, seed users | FR-AUTH-01/02/03/10 | R1 |
| M3 | **Tenancy & onboarding** | Sign-up, tenants, plans (data), memberships, workspace selector, invitations (48 h), plan limits, Super Admin org list | FR-ORG-*, FR-PLAN-*, FR-SA-01 | R1 |
| M4 | **Categories, budgets, expenses (MVP)** | CRUD, Soft limit, 80% alert (once/month), own-only visibility, duplicate warning | FR-CAT, FR-BUD, FR-EXP-01..03, FR-MON-01 | R1 |
| M5 | **Reports & dashboard + FE MVP** | Dashboard, Monthly Summary, notifications; FE: login, wizard, dashboard, add expense, summary, team, categories, budgets, EN/BN switch | FR-RPT, FR-NOT-01 | R1 → **0.1.0** |
| M6 | **Governance** | Hard limit + approval workflow, adjustments, statuses, email/SMS alerts, audit log table/UI | FR-APR, FR-EXP-04/05, FR-NOT-02 | R2 |
| M7 | **Security hardening** | Lockout (3 strikes), forgot/change password, MFA OTP (dev 111111, prod SMS+email, 5-min, countdown), SMS adapters (Mock/FreeGateway/Twilio), `sessionMinutes`, token reuse detection | FR-AUTH-04..08 | R2 |
| M8 | **Permissions delegation** | API + UI page, ceiling & cascade, `PermissionVersion` | FR-AUTH-09 | R2 |
| M9 | **Power tools** | Filter DSL + compiler (EF/in-memory), FE filter bar, saved filters; generic CSV/XLSX import with full error report; async worker | FR-SRCH, FR-BULK | R2 → **0.2.0** |
| M10 | **Scale & portability** | Dapper path + dialects, SQL Server/Postgres/MySQL/Oracle/Mongo/Access, pooling, provider contract tests, backup API | FR-DB-01, FR-OPS-01 | R3 |
| M11 | **Cache & resilience** | HybridCache+Redis backplane, fallback, Memcached adapter, Polly pipelines, aggregates/snapshots, rate limiting, forwarded-IP | FR-CACHE-01, FR-RES-01, FR-AUTH-11, NFR-01 | R3 → **0.3.0** |
| M12 | **PWA & offline** | manifest.json, service-worker.js, install prompt, Dexie cache, outbox sync, conflict UI, offline banner | FR-PWA-01, FR-OFF-01 | R4 |
| M13 | **Assistant** | Engine, intents EN/BN, guardrails, text + voice UI, tests for tenant/role leakage | FR-ASST-01 | R4 → **0.4.0** |
| M14 | **Quality gates** | Performance, load, stress projects; coverage gates; a11y; security scan | FR-TEST-01, NFR-* | R5 |
| M15 | **DevOps & docs** | Compose + nginx LB, rate limit at edge, CI/CD workflows, Postman suite complete, Backend/Frontend guides, ADRs, C4, README polish | FR-DEV-01, FR-DOC-02 | R5 |
| M16 | **1.0 release** | Full regression, release notes, handover | all | **1.0.0** |

## Critical path & risks
`M1 → M2 → M3 → M4` is strictly sequential. M10 (providers) can start after M4 using contract tests while M5–M9 proceed. Highest risk: Oracle/MySQL EF provider compatibility with .NET 10 (check on M10 day 1), MS Access (Windows-only), Bengali voice recognition (browser-dependent), free SMS gateway reliability.

## Task template (copy per slice)
```
Slice: <Area>/<UseCase>      Req: FR-…       Release: R…
1. Domain change?   2. Command/Query + Validator + Handler   3. Endpoint + permission + OpenAPI
4. Persistence (repo method/read model; Dapper SQL per dialect if needed)
5. Messages EN/BN   6. Tests (unit, integration incl. isolation)   7. Postman + Scalar
8. FE (api hook, page, strings)   9. Docs/Release notes/Handover   10. Commit (conventional)
```

## Commit convention
`type(scope): summary` — types `feat|fix|refactor|test|docs|chore|ci|perf`. Small, one concern. Author identity is in `CLAUDE.md`.

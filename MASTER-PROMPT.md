# MASTER-PROMPT — Build Outlay

> Paste everything below the line into an AI coding agent (with repo access) **after** placing this docs folder at the repo root. The agent must read the docs first, work milestone by milestone, and never skip tests.

---

## ROLE
You are a senior full-stack engineer and architect. You build **Outlay**, a multi-tenant Expense Tracking & Budget Control SaaS, exactly as specified in this repository's documents. You write simple, readable, maintainable code (OOP/OOAD, SOLID, KISS, YAGNI, DRY) and you never leave the repo in a broken state.

## READ FIRST (in order)
1. `CLAUDE.md` — standing rules (the "AI brain").
2. `AI-HANDOVER.md` — current state + the exact next command.
3. `MASTER-SPECIFICATION.md` — requirement IDs, rules, decisions (**wins on conflict**).
4. `PRD.md`, `AppFlow.md`, `UiSpec.md`, `TRD.md`, `Schema.md`, `Plan.md`.
5. `docs/adr/*`, `docs/architecture/C4-DIAGRAMS.md`, developer guides.

## NON-NEGOTIABLE CONSTRAINTS
**Architecture**: Clean Architecture + Vertical Slice + CQRS via MediatR (behind our own `ICommand/IQuery`). Layer rules enforced by architecture tests. Backend **.NET 10** (central package mgmt, `global.json`, easy bump to 11+). Frontend **Next.js latest + React 19+**, TypeScript strict, **Tailwind CSS**.
**Auth**: OAuth2/OIDC with **OpenIddict**, JWT, rotating refresh tokens + reuse detection + revocation, RBAC with permission delegation (ceiling + cascade), BCrypt password hashing, 3-strike lockout, MFA OTP (dev `111111`; prod random 6-digit, 5-minute expiry, countdown, SMS+email; free-gateway adapter + Twilio configurable), optional `sessionMinutes` on the login **API only**.
**Persistence**: provider switch by config/env — InMemory (default), SQL Server, MySQL, Oracle, PostgreSQL, MS Access (Windows/Dapper only), MongoDB; ORM switch EF Core ↔ Dapper; pooling; the same contract-test suite runs on every provider.
**Platform**: Redis + HybridCache with graceful fallback (memory/memcached/none); Polly v8 resilience configurable; `Idempotency-Key`; `X-Correlation-Id`; rate limiting; forwarded-IP handling; graceful dependency-failure responses with configurable support contact (name/phone/email).
**Logging**: custom middleware + five configurable categories (`application-logs`, `application-error-logs`, `application-exceptions-logs`, `application-query-logs`, `application-audit-logs`), file `dd-MM-yyyy.txt`, each toggled in `appsettings.json` or `.env`; entries carry class, file name/location, method, line; exceptions include **exact reason + possible fixes**; query logs include SQL, start/end, duration, slow-threshold flag shown in **red**; audit logs include user **IP**; **never** log passwords, tokens, OTPs, secrets, money amounts.
**Every CRUD** gets: advanced multi-field search (contains/startsWith/endsWith/notContains/…, AND/OR), CSV + Excel bulk import with all-errors-at-once and full rollback, localized pagination, permission checks, tenant filter, audit, OpenAPI title/description, Postman request.
**Frontend**: offline-first PWA (`manifest.json`, `service-worker.js`, install prompt), IndexedDB cache + outbox sync, English/Bengali full-screen localization (titles, numerals, pagination), beautiful responsive "Emerald Ledger" design (see `UiSpec.md`), voice + text assistant.
**Ops APIs**: DB backup download for the active provider (Super Admin), log download by category + `dd-MM-yyyy`, release-notes API (from `RELEASENOTES.md`), permissions API + UI.
**Docs/Tests/DevOps**: Scalar docs with a colourful custom theme; Postman collection with pre/post-request scripts per role (Super Admin, Admin, Manager, Member); separate test projects (unit, integration, performance, load, stress, architecture) + FE unit/e2e; Docker multi-stage, compose with nginx load balancer; GitHub Actions CI/CD; README; Backend and Frontend developer guides; ADRs; C4 + ER diagrams.

## WORKING METHOD (repeat per slice)
1. Open `Plan.md`, pick the next unchecked slice in the current milestone.
2. Write/adjust tests first for business rules (red → green).
3. Implement the slice in one feature folder; reuse generic kits (search, import, CRUD UI).
4. Add EN+BN strings, OpenAPI metadata, Postman request + test script.
5. Run: `dotnet build && dotnet test` and `pnpm lint && pnpm typecheck && pnpm test` — **all green**.
6. Update `RELEASENOTES.md` (Unreleased → "Added/Changed/Fixed" + "QA: verify") and `AI-HANDOVER.md` (state + exact next command).
7. Commit with a conventional message using the author identity in `CLAUDE.md`. Small commits.
8. Stop-and-report if a requirement conflicts or is ambiguous: propose options, default to the decision table in `MASTER-SPECIFICATION.md §9`.

## ANTI-REGRESSION RULES
* Never edit a green test to make it pass; fix the code or raise the conflict.
* Never weaken tenant filters, permission checks, redaction, or lockout to "unblock" something.
* Never commit secrets; `.env.example` only.
* Don't add dependencies without a one-line justification in the PR/commit body; prefer the stack already chosen.
* Keep public contracts stable (API v1); breaking changes need an ADR + release note.

## SAFETY AGAINST DRIFT
If context is running low: finish the current slice to green, update `AI-HANDOVER.md` with **the exact next command**, `RELEASENOTES.md`, `ROADMAP.md` status, commit, then stop.

## FIRST ACTIONS (if repo is empty)
```bash
git clone https://github.com/mtabassum/Outlay && cd Outlay       # verify access/contents first (A-11)
# place docs, then execute Milestone M0 from Plan.md:
dotnet new sln -n Outlay && mkdir -p src tests web docs
```

## ACCEPTANCE (project-level)
All FR-/NFR- IDs in `MASTER-SPECIFICATION.md` mapped to passing tests; isolation suite green on every provider available in CI; Docker `compose up` works from `.env.example`; Scalar and Postman usable per role; docs complete.

# CLAUDE.md — Outlay AI Brain

> Read this file at the start of **every** session. It is the operating manual for any AI agent (and human) working on Outlay.
> If something here is wrong or outdated, fix it in the same PR. Never work around it silently.

## 1. What Outlay is

A multi-tenant **Expense Tracking & Budget Control** SaaS. Organisations sign up, pick a plan (Starter/Pro/Enterprise), invite their team, set budgets per category, and track expenses with Soft/Hard limits, approvals, alerts and reports. Bilingual (English/বাংলা), offline-first PWA, deterministic voice/text assistant.
**Owner:** Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh. **Backend:** .NET 10, ASP.NET Core, EF Core 10, MediatR, OpenIddict. **Frontend:** React 19, TypeScript, Vite, Tailwind v4.

## 2. Source of truth (read before coding)

| Need | Read |
|---|---|
| Where everything is, open questions | `MASTER-SPECIFICATION.md` |
| What to build / business rules (`FR-*`, `BR-*`) | `docs/PRD.md` |
| User flows (`F-*`) | `docs/AppFlow.md` |
| Screens and design tokens (`S-*`) | `docs/UiSpec.md` |
| Architecture, security, logging, engines, API catalogue | `docs/TRD.md` |
| Tables, indexes, provider mapping | `docs/Schema.md` |
| What to do next (`M*-*` tasks, Definition of Done) | `docs/Plan.md` + `AI-HANDOVER.md` |
| Why a decision was made | `docs/adr/` |
| Architecture pictures (C4) | `docs/diagrams/C4-DIAGRAMS.md` |
| How to build a backend slice / frontend screen, step by step | `docs/guides/BACKEND-DEVELOPER-GUIDE.md`, `docs/guides/FRONTEND-DEVELOPER-GUIDE.md` |

**Precedence:** owner decision > PRD > newest ADR > TRD/Schema/UiSpec/AppFlow > Plan. Never invent business rules. If a rule is missing or ambiguous, add it to `MASTER-SPECIFICATION.md §6` and ask.

## 3. Repository map

```
Outlay.slnx                         solution at the repo root (+ global.json, Directory.Build.props, Directory.Packages.props)
src/Backend/Outlay.{Domain, Application, Infrastructure, Api, Persistence.<Provider>}
src/Backend/Outlay.Api/Dockerfile   API image (VS generated, build context = repo root)
src/Frontend/outlay-web/            Vite + React app: src/{app, features/<feature>, shared/{ui,lib,hooks,search}, offline,
                                    locales/{en,bn}, styles} · public/manifest.json · tests/e2e · Dockerfile
tests/Backend/Outlay.*Tests         unit, architecture, integration, performance, load (k6), stress (k6)
postman/  collection + environments     deploy/  nginx, docker-compose, .env.example     docs/  specs, adr/, diagrams/, guides/
```
Verified against commit `b5fc8f3` (initial scaffold). The WeatherForecast template was deleted by the owner on 2026-10-28 (pending push). `Program.cs` still has `AddControllers()`/`MapControllers()`, which M0-01 removes. See TRD §4 for the keep/change table.

## 4. Commands

```bash
# Backend (from the repo root)
dotnet restore Outlay.slnx && dotnet build Outlay.slnx -warnaserror
dotnet test tests/Backend/Outlay.UnitTests
dotnet test tests/Backend/Outlay.ArchitectureTests
dotnet test tests/Backend/Outlay.IntegrationTests  # needs Docker (Testcontainers)
dotnet run --project src/Backend/Outlay.Api --launch-profile https   # https://localhost:7024 (default from launchSettings.json) · Development: InMemory DB, OTP 111111, Scalar at /scalar
#   other port on your machine: add --urls "https://localhost:<port>" (don't edit launchSettings.json)
dotnet ef migrations add <Name> -p src/Backend/Outlay.Persistence.PostgreSql -s src/Backend/Outlay.Api -- --Database:Provider=PostgreSql
# repeat the migration for SqlServer, MySql, Oracle (Tier 1+2) in the same PR

# Frontend
cd src/Frontend/outlay-web
npm ci && npm run dev            # http://localhost:5173 (default); API via VITE_API_PROXY_TARGET (override both in .env.development.local)
npm run lint && npm run typecheck && npm test && npm run test:e2e
npm run gen:api                  # regenerate src/api-types.ts from the running API's OpenAPI
npm run i18n:check               # fails if a key is missing in bn or en

# Full stack
cd deploy && cp .env.example .env    # default host ports: 443 LB, 8025 Mailpit, 5432 Postgres, 6379 Redis (change in .env)
docker compose --profile postgres up -d --scale api=2
k6 run tests/Backend/Outlay.LoadTests/load.js
```

## 5. Architecture rules (enforced by `Outlay.ArchitectureTests`)

- Dependencies: `Api → Application → Domain`; `Infrastructure → Application`; `Persistence.* → Infrastructure`. **Domain references nothing.** Application never references Infrastructure or EF provider packages.
- **Vertical slices:** `Application/Features/<Feature>/<UseCase>/` holds `<UseCase>Command|Query`, `<UseCase>Handler`, `<UseCase>Validator`, DTOs. The endpoint lives in `Api/Endpoints/<Feature>/<UseCase>Endpoint.cs` and contains **no logic**.
- **CQRS:** commands change state and return `Result<T>`. Queries never change state, and they project to DTOs.
- Cross-cutting concerns go in **MediatR behaviours**, never in handlers: Logging → Validation → Authorization → Tenant/Plan → Caching → Transaction.
- Handlers are `internal sealed`, use primary constructors, depend only on `Application/Abstractions`.
- Business outcomes are `Result` errors (`ExpenseErrors.PossibleDuplicate`), not exceptions.
- Rich domain: invariants live in entities (`Expense.Approve(...)`), not in handlers.

## 6. How to add a CRUD slice (checklist)

1. Domain: entity/value objects + xUnit `[Theory]` tests for its rules.
2. EF configuration in `Infrastructure/Persistence/Configurations` (snake_case, tenant index first) + migrations for every Tier 1/2 provider.
3. Application: `Create/Update/Delete/Get/Search` slices; validators with **localised** messages; `IRequirePermission`; `IIdempotentCommand` on creates; `ICacheableQuery`/`IInvalidatesCache` tags (tenant-scoped via `CacheKeys`).
4. Search: register `SearchFieldMap<TEntity>` (whitelisted fields only).
5. Import: implement `IImportDefinition<TRow,TCommand>` that **reuses the create validator**.
6. Endpoints: `.WithName().WithSummary().WithDescription().WithTags().Produces…()` + examples.
7. Permissions: add constants to `PermissionCatalog` and role templates. Run TS generation.
8. Tests: handler unit tests (xUnit + NSubstitute + AwesomeAssertions), integration (happy, 422, 403 per role, **cross-tenant 404**, idempotent replay).
9. Postman: add to role folders + RBAC matrix.
10. Frontend: `features/<x>/` (api, queries with `useOfflineMutation`, Zod schema, page using `DataTable` + `FilterBuilder` + `ImportWizard`), `en` + `bn` strings.
11. Docs: spec sections, `RELEASENOTES.md` Unreleased (+ SQA checklist), `AI-HANDOVER.md`.

## 7. Coding conventions

**C#:** file-scoped namespaces; `sealed` by default; primary constructors for DI; `required`/`init` for DTOs (records); async all the way with `CancellationToken`; no `.Result`/`.Wait()`; no `Task.Run` for I/O; `DateTimeOffset.UtcNow` only via `IClock`; money is `decimal` (never `double`) wrapped in `Money`; guard clauses over nesting; methods ≤ ~30 lines; names say intent (`EnsureCanAddCategory`, not `Check2`). Manual mapping via `ToDto()` extensions (no AutoMapper).
**EF Core:** always `.TagWithCaller()` on queries; `AsNoTracking()` for queries; never `IgnoreQueryFilters()` for the `Tenant` filter outside `Features/Tenants|System|Reports/Platform`; use the execution strategy for transactions; Dapper only via `IDapperExecutor`.
**Logging:** use `logger.AppInfo/AppWarning/AppError(...)` helpers (caller info included). Structured templates, never string interpolation. **Never log** passwords, OTPs, tokens, secrets, emails/phones in plain form, or money amounts. Mark sensitive DTO props `[Sensitive]`.
**TypeScript/React:** strict mode; function components; no `any`; server state in TanStack Query, UI state in Zustand; forms with React Hook Form + Zod; **no hard-coded strings** (i18n keys) and **no hard-coded colours** (theme tokens); format numbers and dates only via `useFormat()`; use components from `shared/ui` before creating new ones.
**Commits:** Conventional Commits with the task ID: `feat(expenses): M2-04 create expense slice`. Small PRs.

## 7a. Ports and URLs

- Never hard-code a port in application code. In docs, state the default port, where it comes from and how to override it. Ports come from `launchSettings.json` (shared default, override with `--urls`), `VITE_API_PROXY_TARGET`/`VITE_DEV_PORT` (override in `.env.development.local`), and `deploy/.env` (compose host ports). Application code uses the relative `/api` base URL.

## 8. Security & multi-tenancy guardrails (never violate)

- Tenant comes **only** from the token (`tenant_id` claim). Ignore any tenant id in bodies, files or assistant text.
- Every expense/report query passes through `IDataScopeService`.
- Approval recipients and approve/reject rights come only from `IApproverResolver` (never "all Managers"). Out-of-scope approve returns 404.
- Expense edits reuse the create validator and `IBudgetMonitor` (budget computed **without** the old amount). An edit must never bypass a Hard limit.
- Cache keys only via `CacheKeys` (always `t:{tenantId}:` prefixed).
- A grantor can grant only permissions they hold. Revocation cascades.
- Passwords: BCrypt via `BcryptPasswordHasher` only. OTPs stored hashed. Dev OTP `111111` only in Development (startup guard must stay).
- Super Admin screens and endpoints never return tenant expense amounts or descriptions.
- Client IP comes from forwarded headers of **known proxies** only.
- Secrets never go in committed `appsettings*.json`. Use `.env` / secrets.

## 9. Testing rules

- **Default test stack (ADR-0017):** xUnit for the .NET test projects (unit, integration, architecture), NSubstitute for test doubles, AwesomeAssertions for assertions, Testcontainers (with WebApplicationFactory) for integration tests, BenchmarkDotNet for performance, k6 for load and stress.
- To use a different library, propose an update to ADR-0017 first. FluentAssertions 8+ requires a paid licence, so check licensing before suggesting it.
- Unit tests live in one project, `Outlay.UnitTests`, in `Domain/`, `Application/` and `Infrastructure/` folders mirroring `src/Backend`.
- Every bug fix starts with a failing test.
- Freeze time with `FakeClock`. Never depend on `DateTime.Now` in tests.
- New endpoint means new integration tests for: success, validation (422), forbidden per role (403), foreign tenant (404), idempotent replay where applicable.

## 10. Configuration quick reference (appsettings.json ⇄ .env with `__`)

`Database:Provider` (InMemory|SqlServer|PostgreSql|MySql|Oracle|Access|MongoDb) · `Cache:L2` (Redis|Memcached|None) · `OutlayLogging:Categories:<X>:Enabled` · `OutlayLogging:Categories:Query:SlowQueryThresholdMs` · `Otp:UseFixedDevelopmentCode` · `Sms:Provider` (Console|TextBelt|Twilio) · `Auth:Lockout:MaxFailedAttempts` (3) · `Auth:Session:Max` · `Resilience:*` · `RateLimiting:*` · `Support:TeamName|Phone|Email` · `Backup:Strategy` (Native|Logical). Full reference: TRD §6–§11, §17.2.

## 11. Definition of Done

See `docs/Plan.md` → Definition of Done. Short version: tests green, architecture tests green, OpenAPI complete, Postman updated, EN+BN strings, no sensitive logs, docs + RELEASENOTES + AI-HANDOVER updated, CI green.

## 12. Session protocol for AI agents

1. Read `AI-HANDOVER.md` → pick the next unchecked task in `docs/Plan.md` (or the one the owner names).
2. Restate the task, its acceptance criteria and the spec sections you'll follow. Ask only if a business rule is genuinely ambiguous.
3. Work test-first in small steps. Run the relevant test projects after each step.
4. Don't widen scope (YAGNI). Note ideas in `AI-HANDOVER.md → Ideas parking lot`.
5. Finish by updating: the Plan checkbox, `RELEASENOTES.md` (Unreleased + SQA checklist), `AI-HANDOVER.md` (state, next step, risks), and any spec you changed.
6. Never claim something works without running it. Report exactly what was run and the result.

## 13. Known pitfalls

- Pooled `DbContext` can't take scoped services in its constructor. Set the tenant on the rented instance (TRD §6.3).
- MongoDB EF provider: no `Select` projections, no migrations, no FKs. Use `ProjectSafe` and the Mongo read store.
- MySQL: Pomelo has no EF Core 10 build. Use `MySql.EntityFrameworkCore` (Oracle).
- MS Access works only on Windows (ACE OLE DB). It's excluded from Docker builds.
- MediatR 13+ requires a licence key (Community or paid). Keep slices coupled only to our marker interfaces.
- HybridCache-style L1 caches go stale across replicas without a backplane. Keep FusionCache's backplane on when Redis is present.
- Text log files can't be "red": the console is ANSI red, files use the `[SLOW QUERY ⚠]` prefix, and the S-22 viewer colours them.

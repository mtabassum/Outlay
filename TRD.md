# TRD — Technical Requirements Document

Backend: **.NET 10** (upgradable to 11+) · Frontend: **Next.js (latest) + React 19+** · IDs refer to `MASTER-SPECIFICATION.md`.

## 1. Architecture overview
**Clean Architecture** (dependency rule inward) + **Vertical Slices** (feature folders) + **CQRS** (MediatR). Principles: SOLID, DRY, KISS, YAGNI. Enforced by architecture tests (NetArchTest).

```
src/
  Outlay.Domain/            entities, value objects, enums, domain events, domain rules (no packages)
  Outlay.Application/       Features/<Area>/<UseCase>/ {Command|Query, Handler, Validator, Dto, Endpoint-contract}
                            Abstractions: IRepository<T>, IUnitOfWork, ICacheService, ISmsSender, IEmailSender, IClock, ICurrentUser, ITenantProvider
                            Behaviors: Correlation, Logging, Validation, Authorization, Idempotency, Caching, Transaction
  Outlay.Infrastructure/    Persistence.{InMemory,EfCore,Dapper,Mongo}, Caching, Messaging(Sms/Email), Identity, Logging, Backup, Import
  Outlay.Api/               Minimal API endpoint groups, middleware, OpenIddict host, Scalar, health, rate limiting
  Outlay.Contracts/ (opt.)  public request/response DTOs shared with OpenAPI
tests/ UnitTests · IntegrationTests · PerformanceTests · LoadTests · StressTests · ArchitectureTests
web/   Next.js app
```
**Rules**: Domain references nothing. Application references Domain only. Infrastructure references Application. Api references all (composition root). A feature slice owns its command, handler, validator and endpoint mapping in **one folder**; cross-slice reuse goes through Application abstractions, never slice-to-slice references.

## 2. Request pipeline (backend)
```
Kestrel → ForwardedHeaders → CorrelationIdMiddleware → OutlayLoggingMiddleware (request log, exception capture)
→ RateLimiter → Auth (OpenIddict validation) → TenantResolutionMiddleware → Localization → Endpoint
   └─ MediatR behaviors: Logging → Validation → Authorization(permission) → Idempotency → Caching(queries) → Transaction(commands) → Handler
```
Errors map to RFC 7807 `ProblemDetails` with `code`, `correlationId`, localized `detail`, and `support` contact when a dependency is down.

## 3. CQRS conventions
* `ICommand<TResult>` / `IQuery<TResult>` marker interfaces wrap MediatR `IRequest` (swap-friendly, A-08).
* Commands mutate, return minimal results (ids/status). Queries never mutate; read models may use Dapper/projections.
* Naming: `CreateExpenseCommand`, `CreateExpenseHandler`, `CreateExpenseValidator`, `GetMonthlySummaryQuery`.
* Domain events (e.g., `ExpenseSaved`) published after commit → handlers for notifications, cache invalidation, aggregates (in-process now; outbox-ready).

## 4. Authentication & authorization
* **OpenIddict** server + validation in the API host. Endpoints: `/connect/token`, `/connect/revocation`, `/connect/introspect` (optional), `/.well-known/openid-configuration`, JWKS.
* Grants: `refresh_token` and custom `urn:outlay:grant:login_ticket` (ADR-0003). Client: first-party `outlay-web` (confidential via Next BFF). Scopes: `api`, `offline_access`.
* Tokens: signed JWT access (default 15 min), rotating refresh (default 7 d; capped by `sessionMinutes`), reuse detection → revoke family. Prod signing/encryption keys from X.509 certificate or secret store; dev ephemeral keys.
* Claims: `sub`, `tid` (tenant), `role`, `pv` (permission version), `sid`. Permissions are **not** embedded; resolved via cached lookup keyed `perm:{tenant}:{user}:{pv}`.
* Authorization: `[HasPermission("expenses:approve")]` → policy → `IPermissionService`. Default-deny fallback policy. Platform vs tenant scope: tenant endpoints require `tid`; Super Admin tokens have none ⇒ 403 on tenant data.
* **Passwords**: BCrypt.Net-Next, work factor from `Security:Bcrypt:WorkFactor` (≥12). Password policy validator; breached-password check optional.
* **Encryption at rest** (non-password secrets): AES-256-GCM via ASP.NET Data Protection with keys persisted (Redis/DB/file) and protected by cert. Note: BCrypt is *hashing*, not encryption — passwords are never reversible.
* **Lockout**: counter on `Users` (`FailedLoginCount`, `LockedUntil`/`IsLocked`); 3rd consecutive failure ⇒ locked; reset on success/reset/unlock. Constant-time responses and generic messages to avoid user enumeration.
* **MFA/OTP**: `OtpChallenges` row `{id,userId,purpose,codeHash,expiresAt(5m),attempts,maxAttempts(5),channel,sentAt}`; code stored hashed (HMAC-SHA256 with server key). `Mfa:DevFixedOtp.Enabled` honoured **only if** `ASPNETCORE_ENVIRONMENT` ∈ {Development, Testing} — startup fails fast if enabled in Production. Resend cooldown 60 s; per-user/IP OTP rate limit.
* **Permission delegation** (ADR-0010): `MembershipPermissions(membershipId, permission, grantedByMembershipId)`. Grant rule: `requested ⊆ grantor.effective` and `grantor.role` outranks target. Revoke ⇒ recursive removal of grants whose `grantedBy` chain depended on it. Bump `PermissionVersion`; cache bust.
* **Idempotency/anti-replay**: login ticket single use (cache `SETNX`), 2 min TTL.

## 5. Multi-tenancy
* Shared database, `TenantId` discriminator on every tenant table (ADR-0005).
* EF Core: global query filter `e => e.TenantId == _tenant.Id` + `SaveChanges` interceptor stamps/validates TenantId. Dapper: all SQL goes through `TenantSql` helper that **requires** `@TenantId` (analyzer/test fails queries lacking it). Mongo: repository base adds `Filter.Eq(TenantId)`. InMemory: same repository base.
* Cache keys always prefixed `t:{tenantId}:`. Log lines carry `tenantId`. Bulk import and assistant resolve tenant from token only.
* Mandatory **isolation test suite**: seed two tenants; every endpoint is called with tenant A token against tenant B ids and must return 404/403.

## 6. Persistence abstraction (FR-DB-01)
```jsonc
"Database": { "Provider": "InMemory", // InMemory | SqlServer | MySql | Oracle | PostgreSql | MsAccess | MongoDb
              "Orm": "EfCore",        // EfCore | Dapper  (ignored for MongoDb)
              "ConnectionString": "",
              "Pooling": { "Enabled": true, "MinSize": 5, "MaxSize": 100 },
              "AutoMigrate": true, "SeedDemoData": true, "CommandTimeoutSeconds": 30 }
```
Env override: `OUTLAY_DATABASE__PROVIDER=PostgreSql`, `OUTLAY_DATABASE__ORM=Dapper`.
* Application depends only on `IRepository<T>`, `IUnitOfWork`, and read-model interfaces (e.g., `IMonthlySummaryReader`). Infrastructure registers one implementation set via `PersistenceModule.Add(provider, orm)` (factory + keyed DI).
* **Support matrix**

| Provider | EF Core | Dapper | Notes |
|---|---|---|---|
| InMemory | ✔ (in-memory store) | ✔ (same store adapter) | Default; single replica; no transactions semantics beyond snapshot (emulated UoW) |
| SQL Server | ✔ | ✔ | `Microsoft.Data.SqlClient` |
| PostgreSQL | ✔ (Npgsql) | ✔ | Recommended production default |
| MySQL | ✔ (Pomelo/Oracle provider — verify .NET 10 support at build) | ✔ (MySqlConnector) | |
| Oracle | ✔ (`Oracle.EntityFrameworkCore`) | ✔ (ODP.NET) | Naming ≤ 30 chars on old versions; use quoted identifiers or short names |
| MS Access | ✘ (no maintained provider) | ✔ (OleDb/ODBC) | **Windows only**, ACE driver; A-06 |
| MongoDB | EF Core Mongo provider *or* official driver | n/a | Document model, tenant filter, transactions need replica set |

* **SQL dialect**: Dapper queries live in `Sql/<Provider>/<feature>.sql` embedded resources with a `ISqlDialect` for paging/quoting/booleans/upsert. Keep SQL minimal & portable; advanced filter → SQL built by `SqlFilterCompiler` per dialect.
* **Migrations**: EF Core migrations per provider assembly (`Migrations.SqlServer`, …) or `EnsureCreated` for dev; Dapper path uses the same schema scripts in `Schema/<provider>/NNN_*.sql` run by a tiny versioned migrator (`SchemaVersions` table). Contract tests run the **same test-suite against every provider** (Testcontainers; Access only on Windows CI).
* **Pooling**: ADO.NET pooling params from config; EF `AddDbContextPool`; Mongo `MaxConnectionPoolSize`.
* **Backups** (FR-OPS-01) via `IBackupProvider` per DB: SQL Server `BACKUP DATABASE` to a server path + stream; PostgreSQL `pg_dump`; MySQL `mysqldump`; Oracle Data Pump (`expdp`) or exported schema script; Access file copy of `.accdb` (closed/shared lock aware); Mongo `mongodump --archive`; InMemory JSON snapshot. CLI tools must exist in the image (documented in Dockerfile). Streams, zip-compressed, never written to a public path, audit-logged, Super Admin only, rate-limited.

## 7. Caching (FR-CACHE-01)
* `ICacheService` over **HybridCache** (L1 in-proc + L2 Redis) with tags for invalidation by `t:{tenant}:expenses`.
* Config: `Cache:Provider = Redis | Memory | Memcached | None`, `Cache:Redis:ConnectionString`, TTL per key class, `Cache:FailSafe=true`.
* **Backplane**: Redis pub/sub notifies other replicas to drop L1 entries; also used by SignalR/notification fan-out if added.
* Read-through for queries marked `ICachedQuery` (key = request type + tenant + normalized params + permission version). Commands invalidate by tag via domain-event handlers.
* **Redis down** ⇒ circuit opens, `Error` log once per interval, automatic fallback to L1/in-memory; `/health/ready` reports `degraded`; no user-facing error. Memcached adapter optional (`EnyimMemcachedCore`).
* **Aggregation for NFR-01**: `DailyCategoryTotals(tenant, category, date, total, count)` updated on expense saved/approved/adjusted (same transaction), plus immutable `MonthlySnapshots` for closed months; approval of an older pending expense recomputes the affected day row (answers the PRD discussion question).

## 8. Resilience (FR-RES-01)
```jsonc
"Resilience": { "Retry": {"MaxAttempts":3,"BaseDelayMs":200,"UseJitter":true},
  "CircuitBreaker": {"FailureRatio":0.5,"MinimumThroughput":10,"BreakDurationSeconds":30},
  "TimeoutSeconds": 10, "Bulkhead": {"MaxParallel":50,"QueueLimit":100} }
```
Pipelines (Polly v8 via `Microsoft.Extensions.Resilience`) named `db`, `redis`, `sms`, `email`, `http`. Transient-only retries; **never retry non-idempotent commands without an idempotency key**. Env: `OUTLAY_RESILIENCE__RETRY__MAXATTEMPTS`.

## 9. Logging & diagnostics (FR-LOG-01) — custom middleware
### 9.1 Configuration
```jsonc
"OutlayLogging": {
  "Enabled": true, "RootPath": "logs", "RetentionDays": 30, "FileNameDateFormat": "dd-MM-yyyy",
  "Console": { "Enabled": true, "Color": true },
  "Categories": {
    "Application": { "Enabled": true, "MinLevel": "Information", "Folder": "application-logs" },
    "Error":       { "Enabled": true, "Folder": "application-error-logs" },
    "Exception":   { "Enabled": true, "Folder": "application-exceptions-logs", "IncludeStackTrace": true, "IncludeAdvice": true },
    "Query":       { "Enabled": true, "Folder": "application-query-logs", "SlowQueryThresholdMs": 200,
                     "HighlightSlowInRed": true, "LogParameterValues": false, "CaptureCallSite": "Always" },
    "Audit":       { "Enabled": true, "Folder": "application-audit-logs" } },
  "Redaction": { "Keys": ["password","pwd","token","otp","secret","authorization","connectionstring","amount","budget","total","balance"], "Mask": "***" } }
```
Override by env: `OUTLAY_LOGGING__ENABLED=false`, `OUTLAY_LOGGING__CATEGORIES__QUERY__ENABLED=false`, `OUTLAY_LOGGING__CATEGORIES__QUERY__SLOWQUERYTHRESHOLDMS=150`. Hot-reload through `IOptionsMonitor` (no restart).

### 9.2 Components (KISS)
* `IOutlayLogger<T>`: `Info/Warn/Error/Exception/Audit` methods capture `[CallerFilePath] [CallerMemberName] [CallerLineNumber]` automatically ⇒ **class, file name, file location, method, line** in every entry.
* `LogEntry` record → bounded `Channel<LogEntry>` → single `LogWriterHostedService` → `CategoryFileWriter` (append, daily file `logs/<folder>/dd-MM-yyyy.txt`, UTC+configured offset, rolls at midnight, retention sweeper). Non-blocking; on overflow drops Debug first and counts drops.
* `OutlayLoggingMiddleware`: logs request start/end (Application), measures duration, sets `AsyncLocal` `LogContext {correlationId, tenantId, userId, clientIp, handler}`, catches **unhandled** exceptions → Exception log + problem response; logs handled 4xx/5xx outcomes to Error log.
* `LoggingBehavior<,>` (MediatR) records handler class/method so queries can be attributed even when the caller stack is gone (async).
* `Redactor`: masks values for configured keys in structured properties **and** scrubs known patterns in free text (bearer tokens, JWTs, connection strings, 6-digit OTP after "otp"). Money: any decimal property tagged `[Sensitive(Money)]` or named in `Keys` ⇒ `***`. Phone/email partially masked (`+88017****123`, `r***@techcorp.com.bd`).
* **Client IP**: `ForwardedHeaders` with `KnownProxies/KnownNetworks` from config (so spoofed `X-Forwarded-For` is ignored); `IClientInfo.Ip` used by Audit and Application logs.

### 9.3 Exception advice (reason + possible fixes)
`ExceptionAdvisor` holds an ordered rule table (`Match(Exception) → {Reason, PossibleFixes[]}`) — e.g.

| Exception | Reason (example) | Possible fixes (examples) |
|---|---|---|
| `DbUpdateConcurrencyException` | Row changed by another user since it was read | Reload entity and retry; verify RowVersion round-trips |
| `SqlException` timeout / `NpgsqlException` | Query exceeded `CommandTimeout` or server unreachable | Check DB up; inspect slow-query log; add index; raise timeout |
| `RedisConnectionException` | Redis unreachable | Start Redis; check `Cache:Redis:ConnectionString`; app is on fallback cache |
| `OpenIddictExceptions.*` / `SecurityTokenException` | Token invalid/expired/reused | Refresh token; check clock skew; verify signing key |
| `FormatException` in import | Cell not parsable | Fix cell format; see error row/column |
| Unknown | "Unclassified — see stack trace" | Search correlation ID across logs |
Rules are data-driven and unit-tested; new rules added without touching middleware (OCP).

### 9.4 Line formats (plain text, one entry per block)
```
[2026-10-28 14:03:11.482 +06:00] [ERROR] [corr=7f3c...] [tenant=3b1e...] [user=9a2d...] [ip=103.xx.xx.xx]
  at   : Outlay.Application.Features.Expenses.CreateExpense.CreateExpenseHandler.Handle()
  file : src/Outlay.Application/Features/Expenses/CreateExpense/CreateExpenseHandler.cs:57
  type : System.InvalidOperationException
  msg  : Category 'Travel' has no active budget for 2026
  reason: …           fixes: 1) … 2) …
  stack: …
```
Query log:
```
[2026-10-28 14:03:11.301] [QUERY] [SLOW 412 ms > 200 ms] [corr=…] [provider=PostgreSql orm=EfCore]
  at     : Outlay.Application.Features.Reports.GetMonthlySummary.GetMonthlySummaryHandler.Handle()
  file   : src/Outlay.Application/Features/Reports/GetMonthlySummary/GetMonthlySummaryHandler.cs:34
  started: 14:03:10.889   ended: 14:03:11.301   duration: 412 ms   rows: 1280
  sql    : SELECT … WHERE "TenantId" = @p0 AND …     params: @p0=*** (values hidden)
```
Slow entries are written to console in **red** (ANSI) and prefixed `[SLOW]` in file so they are `grep`-able; fast ones in default/green. Capture points: EF Core `DbCommandInterceptor`, Dapper via `ProfiledDbConnection` decorator, Mongo `CommandStartedEvent/Succeeded/Failed` subscribers. `CaptureCallSite`: `Always` (walk `StackTrace` to first `Outlay.*` frame; needs portable PDBs shipped), `Handler` (cheap AsyncLocal only), or `Off`.

### 9.5 Audit log
Fields: time, correlationId, userId, tenantId, **ip**, userAgent, action (`expense.create`, `auth.login.failed`, `permission.grant`…), entity type/id, outcome, changed field **names** (values redacted per 9.2). Also persisted to `AuditLogs` table (queryable by Admin with `audit:view`).

### 9.6 Log download API
`GET /api/ops/logs?category={application|error|exception|query|audit}&date=dd-MM-yyyy` → file stream (Super Admin, `logs:download`). Validates category (allow-list) and date format; no path input ⇒ no traversal. Audited.

## 10. Idempotency & correlation
* `X-Correlation-Id` (GUID/ULID): accept valid inbound, else generate; echoed; included in logs, ProblemDetails, outbound HTTP/SMS calls.
* `Idempotency-Key` required on `POST` that create resources/imports (optional elsewhere). Table `IdempotencyKeys(key, tenantId, userId, route, requestHash, status, responseCode, responseBodyRef, createdAt, expiresAt)`. Same key+same hash ⇒ replay stored response (`Idempotent-Replayed: true`); same key+different hash ⇒ 422; in-flight ⇒ 409 with `Retry-After`. Fast path via cache, durable via DB. TTL 24 h.

## 11. Advanced search (FR-SRCH-01)
Request:
```json
POST /api/{resource}/search
{ "logic":"and", "filters":[
   {"field":"description","op":"contains","value":"taxi"},
   {"field":"amount","op":"gte","value":1000},
   {"group":{"logic":"or","filters":[{"field":"status","op":"in","value":["Saved","OverBudget"]},{"field":"memberName","op":"startsWith","value":"Fah"}]}} ],
  "sort":[{"field":"date","dir":"desc"}], "page":1, "pageSize":20, "savedFilterId":null }
```
* Each entity declares a `FilterSchema` (allowed fields, types, operators, sortable) → drives validation, OpenAPI, **and the frontend filter builder** (`GET /api/{resource}/filter-schema`).
* `IFilterCompiler`: EF → `Expression<Func<T,bool>>`; Dapper → parameterised SQL per dialect (`LIKE` with escaped wildcards, case-insensitive per provider); Mongo → `FilterDefinition`. Never string-concatenate values; field names only from the allow-list.
* Adding search to a new CRUD = declare schema + inherit `SearchQuery<T>` (≈ 15 lines).

## 12. Bulk import (FR-BULK-01/02)
* Endpoint per CRUD: `GET /{resource}/import/template?format=csv|xlsx`, `POST /{resource}/import` (multipart, `Idempotency-Key`), `GET /imports/{jobId}`, `GET /imports/{jobId}/errors.csv`.
* Parsing: CsvHelper (CSV), ClosedXML or Sylvan (xlsx) — streaming, size/row caps (`Import:MaxRows`, `MaxFileMb`), formula injection neutralised (`=,+,-,@` prefixed on export), MIME + magic-byte check.
* `IImportDefinition<TRow,TEntity>`: columns, per-row validator (FluentValidation), uniqueness keys, mapper. Pipeline: **parse → row validation → in-file duplicate detection → DB duplicate detection (one batched query) → collect ALL errors → if none, single transaction bulk insert (provider-optimal: `SqlBulkCopy`, `COPY`, batched multi-row inserts, `InsertMany`) → invalidate cache**. Any error ⇒ rollback (nothing persisted).
* Error item: `{row, column, code, message(localized), value(redacted if sensitive), severity}`; multiple errors per row/column are all listed (e.g., duplicate + invalid length). Error report downloadable as CSV with original row number + original data + error column so the user fixes and re-uploads the same file.
* Executes on `Channel<ImportJob>` + `ImportWorkerService` (bounded parallelism `Import:MaxParallelJobs`) — **off the request thread**; progress via polling (SSE optional). Plan limits (users, categories) are enforced across the whole batch before insert.

## 13. Assistant (FR-ASST-01) — deterministic & tenant-safe
* `POST /api/assistant/query {text, locale}` → `AssistantEngine`: normalise → `IntentMatcher` (lexicon + regex for EN/BN; period parser: today, yesterday, this/last month, quarter, "June", "জুন"; entity resolver for category/member **restricted to the caller's tenant & visible users**) → `Intent` → mapped to an existing MediatR query → templated localized answer (+ optional table rows).
* Intents v1: `SpendTotal`, `TopCategory`, `TopMember`, `BudgetRemaining`, `OverBudgetForecast` (linear run-rate), `ListExpenses`, `CompareMonths`, `PendingApprovals`, `ExpenseCountByMember`, `SeatUsage`, `Help`.
* **Guardrails**: (1) tenant from token only; (2) every intent goes through the same permission policies/queries as the UI — assistant has **no repository access of its own**; (3) visibility by role: Admin → whole tenant (managers + members), Manager → team per `reports:view-team`, Member → self; Super Admin → platform stats only; (4) names resolved only within visible set — otherwise reply "I can't share that information" identical for *not found* and *not allowed*; (5) no data from other tenants or Super Admins ever; (6) no write intents; (7) audit entry per query (intent + params, no amounts); (8) prompt-injection-proof by construction (no LLM); (9) rate limited.
* Frontend voice: Web Speech API (`SpeechRecognition`, `speechSynthesis`) with `lang` `en-US` / `bn-BD`; capability detection; text path always works. Optional server STT/TTS adapter later.

## 14. Notifications & messaging
* In-app (`Notifications` table; unread counts cached). Email via `IEmailSender` (SMTP/MailKit; Mailpit in dev). SMS via `ISmsSender` with providers `Mock | FreeGateway | Twilio` (`Sms:Provider`). `FreeGateway` adapter targets a configurable HTTP endpoint (TextBee/Textbelt-style) — **verify provider terms/availability** (A-04). Twilio: `Sms:Twilio:AccountSid/AuthToken/From` from env/secret store.
* Alert de-dup: `AlertLedger(tenantId, categoryId, kind, periodKey)` unique index ⇒ "once per period".
* Sending happens after commit, with resilience pipeline `sms`/`email`; failures logged and retried by a background outbox (`OutboxMessages`).

## 15. Graceful degradation (FR-DEG-01)
`DependencyGuard` classifies exceptions (DB, Redis, SMS, SMTP, timeout) → `DependencyUnavailableException` → ProblemDetails:
```json
{ "type":"urn:outlay:problem:dependency-unavailable","title":"Service temporarily unavailable","status":503,
  "code":"DEP_DB_DOWN","detail":"<localized>","correlationId":"…",
  "support":{"name":"Outlay Support","phone":"+8801XXXXXXXXX","email":"support@outlay.example"} }
```
`Support:Name|Phone|Email` from appsettings/env. Frontend maps `code` → localized message + `SupportContactCard`. Health: `/health/live` (process), `/health/ready` (DB, Redis[degraded OK], SMTP, SMS).

## 16. API standards
* Minimal API groups per feature: `app.MapGroup("/api/expenses").WithTags("Expenses")`. Versioning: URL `/api/v1/...` from day one.
* **Every endpoint**: `.WithName()`, `.WithSummary("Title")`, `.WithDescription(...)`, response types, `Produces/ProducesProblem`, permission metadata, examples → OpenAPI (built-in `Microsoft.AspNetCore.OpenApi`).
* **Scalar** (`Scalar.AspNetCore`) at `/scalar`: custom colourful theme (emerald/violet/amber gradients, branded header, dark & light), grouped tags, auth pre-configured for OpenIddict password-less flow (token pasted from Postman/login). Disabled or auth-gated in Production via config.
* Postman: `postman/Outlay.postman_collection.json` + environments `local`, `dev`, `prod`. Folders by feature; **collection-level pre-request script** obtains/refreshes token for the active role variable (`role` = superadmin|admin|manager|member), sets `X-Correlation-Id` (uuid) and `Idempotency-Key` for POSTs; **test scripts** assert status/schema and store ids (`expenseId`, `jobId`…). Role-based credentials in environment files (dev only).
* Release notes API: `GET /api/v1/release-notes`, `/latest`, `/{version}` — parsed from `RELEASENOTES.md` (strict heading format) at startup and cached.
* Pagination envelope `{items,total,page,pageSize}`; money serialised as strings/decimals with explicit currency; dates ISO-8601 UTC.
* Rate limiting: fixed + token-bucket policies (`global`, `per-user`, `auth` strict: e.g., 5/min/IP on login & OTP, `import` low) with `Retry-After`; edge `limit_req` in nginx too.
* CORS allow-list from config; HSTS; security headers (CSP strict for web); request size limits.

## 17. Localization (backend)
`Accept-Language` / `?lang=` → `RequestLocalization` (`en`, `bn`). Resources in `Resources/Messages.{en,bn}.json` keyed by stable codes (`EXP_AMOUNT_POSITIVE`). Validators, ProblemDetails, emails, SMS templates, import errors, assistant answers all use keys. Missing key ⇒ English + warning log. Unit test fails if `en` and `bn` key sets differ.

## 18. Frontend architecture (see `docs/FRONTEND-DEVELOPER-GUIDE.md`)
* **Next.js App Router** (latest stable, React 19+ with React Compiler where stable), TypeScript strict, Tailwind v4, `next-intl`, TanStack Query (+ persister to IndexedDB), React Hook Form + Zod, Dexie (IndexedDB), lucide-react. Feature folders `src/features/<name>/{api,components,hooks,schemas,pages}`; shared `src/components/ui`.
* **BFF**: Next route handlers `/api/session/*` proxy auth to the backend; refresh token in `httpOnly; Secure; SameSite=Lax` cookie; access token held in memory only; CSRF via same-site + custom header; no tokens in `localStorage`/IndexedDB.
* **Offline**: Dexie tables per entity (per-user, per-tenant DB name; wiped on logout) + `outbox(id, method, url, body, idempotencyKey, createdAt, attempts, status)`; sync engine with exponential backoff; conflicts → UI. Reads use TanStack Query persisted cache (stale-while-revalidate).
* **Generic CRUD kit**: `createCrudFeature({resource, columns, filterSchema, formSchema, importDefinition})` produces list+search+form+import UI so adding a CRUD is configuration (DRY; Guide §).
* **Upgradeability**: no deprecated/experimental APIs where avoidable, no framework patches, dependencies pinned via lockfile + Renovate/Dependabot; `engines` field; CI job "latest" builds against `next@latest` weekly (allowed-fail).
* Env: `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_SUPPORT_NAME/PHONE/EMAIL`, `NEXT_PUBLIC_ENV`.

## 19. DevOps
* **Docker**: multi-stage `backend/Dockerfile` (SDK → publish → `aspnet` runtime, non-root, includes portable PDBs, DB CLI tools via build arg), `web/Dockerfile` (standalone output). `.dockerignore`.
* **Compose**: `compose.yml` (api×N, web, nginx LB, redis, postgres), profiles `mssql|mysql|oracle|mongo|mail|dev`. `nginx.conf`: upstream least_conn, `limit_req_zone`, health checks, gzip, TLS termination (certs mounted), forwards `X-Forwarded-*`. InMemory profile runs **one** API container (A-07).
* **.env.example** is copy-and-run: `cp .env.example .env && docker compose up --build`.
* **CI/CD (GitHub Actions)**: `ci.yml` (restore/build/format → unit + architecture tests → integration tests with Testcontainers matrix [postgres, sqlserver, mysql, mongo] → FE lint/typecheck/vitest/build → Playwright smoke → coverage gate → CodeQL + dependency review + Trivy image scan → docker build), `cd.yml` (on tag `v*`: build/push images to GHCR, attach release notes from `RELEASENOTES.md`, deploy via SSH compose or environment gate), `nightly.yml` (performance/load/stress smoke, latest-framework canary). Branch protection requires CI green.
* **Config precedence**: defaults < `appsettings.json` < `appsettings.{Env}.json` < `.env`/environment (`OUTLAY_` prefix) < secret store. Secrets never committed.

## 20. Testing strategy (separate projects)
| Project | Tooling | Purpose |
|---|---|---|
| `Outlay.UnitTests` | xUnit, FluentAssertions, NSubstitute | Domain rules (soft/hard, 80% once, plan limits), validators, advisor rules, redactor, filter compiler |
| `Outlay.IntegrationTests` | WebApplicationFactory, Testcontainers | End-to-end API per provider, tenant isolation, auth/lockout/MFA/token rotation, idempotency, import all-or-nothing, log files written |
| `Outlay.PerformanceTests` | BenchmarkDotNet | Filter compiler, redactor, cache path, summary query |
| `Outlay.LoadTests` | NBomber or k6 | Target RPS & p95 on read/write mixes |
| `Outlay.StressTests` | NBomber/k6 | Spike, soak, Redis/DB failure injection (graceful degradation) |
| `Outlay.ArchitectureTests` | NetArchTest | Layer rules, no cross-slice refs |
| `web` | Vitest + RTL; Playwright e2e (EN/BN, offline, a11y via axe) | UI & flows |

## 21. Security checklist (excerpt)
BCrypt ≥12 · lockout · OTP hashed + expiry + attempt cap · refresh rotation + reuse detection · revocation on password/role change · no tokens in browser storage · CSP/HSTS/headers · input validation everywhere · parameterised SQL only · allow-listed filter fields · file upload validation · no sensitive/money data in logs · tenant isolation tests · rate limits · least-privilege DB user · secrets via env/secret store · dependency + container scans in CI · audit with IP.

## 22. Upgrade path
.NET: change `TargetFramework` (central `Directory.Build.props` + `global.json`), bump `Microsoft.*` via central package management (`Directory.Packages.props`), run full test matrix; avoid preview APIs. Frontend: dependency bump + codemods; weekly canary build. Each upgrade recorded as an ADR + release note.

# Backend Developer Guide

Audience: developers extending the .NET API. Read `TRD.md` for the why; this is the how.

## 1. Solution structure
```
src/
  Outlay.Domain/           Entities, value objects, enums, domain services, domain events
  Outlay.Application/
    Abstractions/          Ports: IRepository<T>, IUnitOfWork, ICacheService, ISmsSender, IClock, ICurrentUser…
    Behaviors/             MediatR pipeline: Logging, Validation, Authorization, Idempotency, Caching, Transaction
    Common/                Result/Error types, FilterSchema, ImportDefinition base classes
    Features/<Area>/<UseCase>/   ← vertical slice (command|query + handler + validator + dto)
  Outlay.Infrastructure/   Persistence.{InMemory,EfCore,Dapper,Mongo}, Caching, Messaging, Identity, Logging, Backup, Import
  Outlay.Api/              Program.cs, Endpoints/*, Middleware/*, OpenIddict host, Scalar, Health
tests/                     UnitTests · IntegrationTests · PerformanceTests · LoadTests · StressTests · ArchitectureTests
```
**Dependency rule**: `Domain ← Application ← Infrastructure ← Api`. Domain has no NuGet references. Architecture tests fail the build otherwise.

## 2. Request lifecycle
HTTP → middleware (forwarded headers, correlation, logging, rate limit, auth, tenant, localization) → endpoint → `ISender.Send(command)` → behaviors (log → validate → authorize → idempotency → cache/transaction) → handler → repository → response/ProblemDetails.

## 3. Configuration & switches
`appsettings.json` → `appsettings.{Env}.json` → env `OUTLAY_*` → secrets. Strongly typed options (`DatabaseOptions`, `OutlayLoggingOptions`, `CacheOptions`, `ResilienceOptions`, `SecurityOptions`, `SmsOptions`, `SupportOptions`) validated at startup with `ValidateOnStart()`. To switch DB: set `Database:Provider`/`Orm`; `PersistenceModule` wires the right repositories — **handlers never know**.

## 4. Create a new CRUD (the recipe — ~30 minutes)
Example: new entity `Vendor`.

1. **Domain** — `Domain/Vendors/Vendor.cs`: private setters, constructor enforcing invariants, behaviour methods (`Rename`, `Deactivate`). Inherit `TenantEntity` (adds `TenantId`, audit, `RowVersion`).
2. **Schema** — add to `Schema.md`; add EF configuration (`VendorConfiguration`) + Dapper SQL in `Sql/<Provider>/vendors.sql` (or rely on the generic repository for simple tables); add migration/schema script.
3. **Slices** under `Application/Features/Vendors/`:
   ```
   CreateVendor/   CreateVendorCommand.cs  CreateVendorHandler.cs  CreateVendorValidator.cs
   UpdateVendor/   …           DeleteVendor/ …
   GetVendor/      GetVendorQuery.cs …     SearchVendors/ SearchVendorsQuery.cs  VendorFilterSchema.cs
   ImportVendors/  VendorImportDefinition.cs
   ```
   * Command: `public sealed record CreateVendorCommand(string Name) : ICommand<VendorId>, IIdempotent, IRequirePermission("vendors:create");`
   * Handler: load/validate → `Vendor.Create(...)` → `repo.Add` → `uow.SaveChanges` (Transaction behavior wraps) → raise event.
   * Validator (FluentValidation) uses **message keys**: `RuleFor(x => x.Name).NotEmpty().WithMessage("VENDOR_NAME_REQUIRED").MaximumLength(50)`.
4. **Search** — implement `VendorFilterSchema : FilterSchema<Vendor>` listing allowed fields/types/operators; `SearchVendorsQuery : SearchQuery<Vendor, VendorDto>`. The generic compiler does the rest (EF/Dapper/Mongo).
5. **Bulk import** — implement `VendorImportDefinition : ImportDefinition<VendorRow, Vendor>`: column map, row validator, unique key (`Name`), mapper. Register with `services.AddImport<VendorImportDefinition>()` → template/upload/status/error endpoints appear automatically.
6. **Permissions** — add `vendors:read|create|update|delete|import` to `PermissionCatalog` (+ min role, scope Tenant) → appears on the Permissions page automatically.
7. **Endpoints** — `Api/Endpoints/VendorEndpoints.cs`:
   ```csharp
   var g = app.MapGroup("/api/v1/vendors").WithTags("Vendors").RequireAuthorization();
   g.MapPost("/", Create).WithName("CreateVendor").WithSummary("Create a vendor")
     .WithDescription("Creates a vendor in the caller's organisation. Requires vendors:create.")
     .Produces<VendorId>(201).ProducesProblem(422);
   ```
   plus `GET /{id}`, `PUT`, `DELETE`, `POST /search`, `GET /filter-schema`, import endpoints (mapped by the kit).
8. **Messages** — add keys to `Resources/Messages.en.json` and `.bn.json` (test fails if they differ).
9. **Audit** — writes automatically audited by the `Transaction` behavior via `IAuditable` on the command (action name `vendor.create`).
10. **Tests** — unit (domain + validator), integration (happy path, validation, **wrong tenant → 404**, **wrong role → 403**, search operators, import all-or-nothing incl. duplicate + invalid rows), add a Postman request with pre/post scripts.
11. **Docs** — `RELEASENOTES.md` Unreleased: Added + "QA: what to verify".

## 5. Logging usage
```csharp
public sealed class CreateVendorHandler(IVendorRepository repo, IOutlayLogger<CreateVendorHandler> log) : ICommandHandler<CreateVendorCommand, VendorId>
{
    public async Task<VendorId> Handle(CreateVendorCommand c, CancellationToken ct)
    {
        log.Info("Creating vendor");           // caller class/file/method/line captured automatically
        ...
        log.Audit("vendor.create", entity: nameof(Vendor), id: vendor.Id);   // → application-audit-logs (+ AuditLogs table)
    }
}
```
* Don't pass amounts/secrets as properties; if unavoidable mark with `[Sensitive]`. Don't catch-and-swallow: let the middleware log exceptions; for handled failures return `Result.Failure(Error)` (→ Error log).
* Add exception advice for new failure modes in `ExceptionAdvisorRules.cs`.
* Slow query threshold tuning: `OutlayLogging:Categories:Query:SlowQueryThresholdMs`.

## 6. Persistence tips
* Handlers use repositories; read-heavy screens use dedicated reader interfaces (`IMonthlySummaryReader`) implemented per ORM.
* Dapper SQL: embedded resource per provider, always `@TenantId`, parameters only, paging via `ISqlDialect`.
* Add a provider contract test to `PersistenceContractTests` when you add a repository method.

## 7. Caching
Mark queries `ICachedQuery` (key parts + TTL) — never cache commands. Invalidate with tags in domain-event handlers. Keys are tenant-prefixed automatically. Assume cache can vanish at any moment.

## 8. Errors & problem details
Throw domain exceptions only for invariant violations; use `Result` for expected failures. Map in `ProblemDetailsFactory`. Dependency failures → `DependencyUnavailableException` (support contact added). Always include `correlationId`.

## 9. Auth & permissions
Use `[HasPermission]`/`IRequirePermission`. Never check roles directly in handlers. Tenant context from `ITenantProvider`. Super Admin endpoints live in `/api/v1/platform/*` with platform permissions.

## 10. Testing
`dotnet test` runs unit + architecture + integration (Testcontainers needs Docker). Filters: `--filter Category=Unit`. Performance/load/stress projects run separately (`dotnet run -c Release --project tests/Outlay.PerformanceTests`). Keep tests deterministic: inject `IClock`, fixed GUID factory, seeded data.

## 11. Upgrading .NET
Change `global.json` + `TargetFramework` in `Directory.Build.props`; bump `Microsoft.*` in `Directory.Packages.props`; build, run all tests, run smoke compose; write ADR + release note.

## 12. Definition of done
See `Plan.md`. Checklist: layers respected · tests · EN/BN · OpenAPI · Postman · logging safe · release note.

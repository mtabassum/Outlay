# Outlay — Backend Developer Guide

| Field | Value |
|---|---|
| Audience | C# / ASP.NET Core developers (and AI agents) working on `src/Backend` |
| Version | 0.0.0 · 2026-10-28 |
| Stack | .NET 10 · ASP.NET Core Minimal APIs · EF Core 10 · MediatR · FluentValidation · OpenIddict · Serilog · FusionCache |
| Read first | [CLAUDE.md](../../CLAUDE.md) (rules) · [TRD.md](../TRD.md) (design) · [Schema.md](../Schema.md) (tables) · [C4-DIAGRAMS.md](../diagrams/C4-DIAGRAMS.md) (pictures) |

> This guide is the practical "how". When it disagrees with the TRD, the TRD wins: fix this guide in the same PR.
> Code samples show the **intended** shape of the codebase. Until milestone M0 lands, some of the referenced base types don't exist yet.

---

## Contents

1. [Getting started](#1-getting-started)
2. [Architecture tour](#2-architecture-tour)
3. [Build a CRUD slice: worked example (Expense Categories)](#3-build-a-crud-slice-worked-example)
4. [Cross-cutting how-tos](#4-cross-cutting-how-tos)
5. [Testing guide](#5-testing-guide)
6. [Databases, migrations and providers](#6-databases-migrations-and-providers)
7. [Debugging playbook](#7-debugging-playbook)
8. [Upgrading .NET and packages](#8-upgrading-net-and-packages)
9. [Pull-request checklist](#9-pull-request-checklist)

---

## 1. Getting started

### 1.1 Prerequisites

| Tool | Version | Why |
|---|---|---|
| .NET SDK | 10.0.x (pinned in `global.json`) | Build and run |
| Docker Desktop / Engine | Current | Testcontainers, docker-compose, Redis, databases |
| IDE | Visual Studio 2026 / Rider / VS Code + C# Dev Kit | VS uses the existing "Container (Dockerfile)" launch profile |
| EF Core tools | `dotnet tool install --global dotnet-ef` (version matching EF Core 10) | Migrations |
| k6 | Latest | Load and stress tests |
| Test libraries | Restored by NuGet: xUnit v3, NSubstitute, AwesomeAssertions, Testcontainers, NetArchTest, BenchmarkDotNet | See ADR-0017 |
| Postman or Newman | Latest | API collections |

### 1.2 First run (no database needed)

```bash
git clone https://github.com/mtabassum/Outlay.git && cd Outlay
dotnet restore Outlay.slnx
dotnet run --project src/Backend/Outlay.Api --launch-profile https
```

- API: `https://localhost:7024` (`https` profile) or `http://localhost:5071` (`http` profile). These defaults come from `src/Backend/Outlay.Api/Properties/launchSettings.json`. Scalar API reference: `https://localhost:7024/scalar`. OpenAPI JSON: `https://localhost:7024/openapi/v1.json`. The console also prints the address as `Now listening on: …`.
- **Port taken or different on your machine?** Don't edit `launchSettings.json` (it's shared with the team). Override for yourself instead:
  `dotnet run --project src/Backend/Outlay.Api --urls "https://localhost:<port>"`, or set the `ASPNETCORE_URLS` environment variable. Then point the frontend at that port with `VITE_API_PROXY_TARGET` (Frontend Developer Guide §1.4).
- Development defaults: `Database:Provider=InMemory`, demo data seeded, **OTP is always `111111`**, SMS and email go to the console.
- Demo accounts (Development only, password `Outlay@Dev2026!`):

| Role | Email |
|---|---|
| Super Admin | `superadmin@outlay.app` |
| Admin (TechCorp BD) | `rahim@techcorp.com.bd` |
| Manager | `nusrat@techcorp.com.bd` |
| Member | `fahmida@techcorp.com.bd` |

### 1.3 Full stack with Docker

```bash
cp deploy/.env.example deploy/.env           # edit passwords; change host ports here if any are taken
docker compose -f deploy/docker-compose.yml --profile postgres up -d --scale api=2
# nginx LB: https://localhost  ·  Mailpit (emails): http://localhost:8025
docker compose -f deploy/docker-compose.yml ps   # shows the host port mapped for each service
```

| Service | Default host port | Set by (in `deploy/.env`) |
|---|---|---|
| Load balancer (app + API) | `443` (HTTPS), `80` (HTTP) | `LB_HTTPS_PORT`, `LB_HTTP_PORT` |
| Mailpit web UI | `8025` | `MAILPIT_UI_PORT` |
| PostgreSQL | `5432` | `POSTGRES_PORT` |
| SQL Server / MySQL / Oracle / MongoDB | `1433` / `3306` / `1521` / `27017` | `SQLSERVER_PORT` / `MYSQL_PORT` / `ORACLE_PORT` / `MONGO_PORT` |
| Redis | `6379` | `REDIS_PORT` |

`docker-compose.yml` maps each one as `"${VAR:-default}:<container port>"`. If a port is already used on your machine, change it in your own `deploy/.env` (git-ignored). Ports *inside* the Docker network (e.g. `postgres:5432` in connection strings) never change.

### 1.4 Switching the database

Change one setting. Nothing else in the code changes.

```dotenv
# deploy/.env  (or user secrets / appsettings.Development.json)
Database__Provider=SqlServer
Database__ConnectionStrings__SqlServer=Server=localhost,1433;Database=Outlay;User Id=sa;Password=<pwd>;TrustServerCertificate=True
```

Then start that database's compose profile: `--profile sqlserver`. Supported values and their limits are in TRD §6.2.

### 1.5 Useful configuration switches while developing

| Want to… | Set |
|---|---|
| See every SQL statement | `OutlayLogging__Categories__Query__LogAllQueries=true` |
| Lower the slow-query threshold | `OutlayLogging__Categories__Query__SlowQueryThresholdMs=50` |
| See query parameter values (Development only) | `OutlayLogging__Categories__Query__IncludeParameters=true` |
| Run without Redis | `Cache__L2=None` |
| Turn a log category off | `OutlayLogging__Categories__Audit__Enabled=false` |
| Real OTPs locally | `Otp__UseFixedDevelopmentCode=false` and `Sms__Provider=Console` (the code prints masked; read it from Mailpit) |

---

## 2. Architecture tour

### 2.1 The dependency rule

```
Outlay.Api ───────────────► Outlay.Application ───► Outlay.Domain
    │                             ▲
    └──► Outlay.Infrastructure ───┘
              ▲
Outlay.Persistence.<Provider> ──┘
```

- **Domain:** entities, value objects, domain events, business rules. No NuGet packages and no EF attributes.
- **Application:** features (vertical slices), pipeline behaviours, and the **abstractions** Infrastructure implements (`IAppDbContext`, `IAppCache`, `ISmsSender`…).
- **Infrastructure:** EF Core, Identity/OpenIddict, Serilog, FusionCache, SMS/email, background workers, file readers.
- **Persistence.<Provider>:** registers one DB provider and holds its migrations.
- **Api:** composition root, endpoints, middleware, SignalR hubs, OpenAPI/Scalar.

`Outlay.ArchitectureTests` fails the build if a reference breaks this rule, or if one feature folder references another feature's internals.

### 2.2 What happens on a request

```
HTTP → ForwardedHeaders → CorrelationId → RequestLogging → GlobalException → Localization → RateLimiter
     → Authentication/Authorization → Endpoint (+IdempotencyFilter) → ISender.Send(command)
     → LoggingBehavior → ValidationBehavior → AuthorizationBehavior → TenantGuard/PlanLimitBehavior
     → CachingBehavior (queries) | TransactionBehavior (commands) → Handler → Domain → DbContext
```

If you need something to happen on **every** request or command, it belongs in a middleware or behaviour, **not** in a handler.

### 2.3 Where things live

```
src/Backend/Outlay.Application/Features/Categories/
├── CreateCategory/
│   ├── CreateCategoryCommand.cs
│   ├── CreateCategoryHandler.cs
│   └── CreateCategoryValidator.cs
├── UpdateCategory/ …
├── DeactivateCategory/ …
├── GetCategory/ …
├── SearchCategories/ …
├── ImportCategories/ …
├── CategoryDto.cs              # shared by the slices of this feature only
└── CategoryMappings.cs         # ToDto() extension methods (ADR-0016)

src/Backend/Outlay.Api/Endpoints/Categories/
├── CreateCategoryEndpoint.cs
└── …                           # one file per use case, mirrors Application
```

**Rule of thumb:** a change to "categories" should touch only `Features/Categories`, `Endpoints/Categories`, the EF configuration, and tests.

### 2.4 Building blocks you'll use every day

```csharp
// Application/Common/Messaging
public interface ICommand<TResponse> : IRequest<TResponse>;
public interface IQuery<TResponse>   : IRequest<TResponse>;
public interface ICommandHandler<TCommand, TResponse> : IRequestHandler<TCommand, TResponse> where TCommand : ICommand<TResponse>;
public interface IQueryHandler<TQuery, TResponse>     : IRequestHandler<TQuery, TResponse>   where TQuery   : IQuery<TResponse>;

// Opt-in markers read by behaviours
public interface IRequirePermission     { string Permission { get; } }
public interface ITransactionalCommand  { }
public interface IIdempotentCommand     { }
public interface ICacheableQuery        { string CacheKey { get; } IReadOnlyList<string> Tags { get; } TimeSpan? Duration => null; }
public interface IInvalidatesCache      { IReadOnlyList<string> TagsToInvalidate { get; } }

// Domain/Common
public sealed record Error(string Code, string MessageKey, ErrorType Type, object? Args = null);
public enum ErrorType { Validation, NotFound, Conflict, Forbidden, Unauthorized }
public readonly record struct Result<T> { /* IsSuccess, Value, Error; implicit conversions from T and Error */ }
```

---

## 3. Build a CRUD slice: worked example

We'll build **Expense Categories** (PRD FR-CAT, BR-CAT-01…05, task M2-02) end to end. Every other CRUD follows the same 11 steps (CLAUDE.md §6).

### Step 1: Domain entity + rules

```csharp
// src/Backend/Outlay.Domain/Categories/ExpenseCategory.cs
namespace Outlay.Domain.Categories;

public sealed class ExpenseCategory : AggregateRoot, ITenantOwned, IAuditable
{
    public const int NameMaxLength = 50;

    private ExpenseCategory() { }                       // EF

    public Guid TenantId { get; private set; }
    public string Name { get; private set; } = default!;
    public string NormalizedName { get; private set; } = default!;
    public string? Description { get; private set; }
    public bool IsActive { get; private set; } = true;
    public DateTimeOffset? DeactivatedAt { get; private set; }

    public static Result<ExpenseCategory> Create(Guid tenantId, string name, string? description)
    {
        var nameResult = CategoryName.Create(name);          // value object enforces BR-CAT-02
        if (nameResult.IsFailure) return nameResult.Error;

        return new ExpenseCategory
        {
            Id = Guid.CreateVersion7(),
            TenantId = tenantId,
            Name = nameResult.Value.Display,
            NormalizedName = nameResult.Value.Normalized,
            Description = description?.Trim()
        };
    }

    public Result<ExpenseCategory> Rename(string name)
    {
        var nameResult = CategoryName.Create(name);
        if (nameResult.IsFailure) return nameResult.Error;
        (Name, NormalizedName) = (nameResult.Value.Display, nameResult.Value.Normalized);
        return this;
    }

    public void Deactivate(DateTimeOffset now) { if (!IsActive) return; IsActive = false; DeactivatedAt = now; }
    public void Reactivate()                   { IsActive = true; DeactivatedAt = null; }
}
```

```csharp
// src/Backend/Outlay.Domain/ValueObjects/CategoryName.cs
public sealed record CategoryName(string Display, string Normalized)
{
    public static Result<CategoryName> Create(string? raw)
    {
        var display = Regex.Replace(raw ?? "", @"\s+", " ").Trim();
        if (display.Length == 0) return CategoryErrors.NameRequired;
        if (display.Length > ExpenseCategory.NameMaxLength) return CategoryErrors.NameTooLong;
        return new CategoryName(display, display.ToUpperInvariant());
    }
}
```

**Domain unit tests: xUnit `[Theory]` tables** (`tests/Backend/Outlay.UnitTests/Domain/Categories/CategoryNameTests.cs`):

```csharp
public sealed class CategoryNameTests
{
    [Theory]
    [InlineData("Travel", "TRAVEL")]
    [InlineData("  travel   & transport ", "TRAVEL & TRANSPORT")]
    [InlineData("খাবার", "খাবার")]                       // Bengali stays intact
    public void Create_ValidName_Normalises(string input, string expected)
        => CategoryName.Create(input).Value.Normalized.Should().Be(expected);

    [Theory]
    [InlineData("")]
    [InlineData("   ")]
    [InlineData(null)]
    public void Create_Blank_Fails(string? input)
        => CategoryName.Create(input).Error.Should().Be(CategoryErrors.NameRequired);

    [Fact]
    public void Create_51Chars_Fails()
        => CategoryName.Create(new string('a', 51)).Error.Should().Be(CategoryErrors.NameTooLong);
}
```

`.Should()` comes from **AwesomeAssertions** (`using AwesomeAssertions;`), the free fork of FluentAssertions (ADR-0017). Don't add the paid FluentAssertions 8+.

Errors live next to the entity. Message keys point to the localisation resources (§4.9):

```csharp
public static class CategoryErrors
{
    public static readonly Error NameRequired  = new("category.name_required",  "Category.NameRequired",  ErrorType.Validation);
    public static readonly Error NameTooLong   = new("category.name_too_long",  "Category.NameTooLong",   ErrorType.Validation);
    public static readonly Error DuplicateName = new("category.duplicate_name", "Category.DuplicateName", ErrorType.Conflict);
    public static readonly Error NotFound      = new("category.not_found",      "Category.NotFound",      ErrorType.NotFound);
}
```

### Step 2: EF Core configuration

```csharp
// src/Backend/Outlay.Infrastructure/Persistence/Configurations/ExpenseCategoryConfiguration.cs
internal sealed class ExpenseCategoryConfiguration : IEntityTypeConfiguration<ExpenseCategory>
{
    public void Configure(EntityTypeBuilder<ExpenseCategory> b)
    {
        b.ToTable("expense_categories");
        b.HasKey(x => x.Id);
        b.Property(x => x.Name).HasMaxLength(ExpenseCategory.NameMaxLength).IsRequired();
        b.Property(x => x.NormalizedName).HasMaxLength(ExpenseCategory.NameMaxLength).IsRequired();
        b.Property(x => x.Description).HasMaxLength(250);

        b.HasIndex(x => new { x.TenantId, x.NormalizedName }).IsUnique()
         .HasDatabaseName("ux_expense_categories_tenant_normalized_name");      // BR-CAT-01
        b.HasIndex(x => new { x.TenantId, x.IsActive })
         .HasDatabaseName("ix_expense_categories_tenant_active");
    }
}
```

You **don't** add the tenant or soft-delete filters here. `OutlayDbContext.OnModelCreating` applies the named `Tenant` and `SoftDelete` filters to every `ITenantOwned`/`ISoftDeletable` entity automatically. Concurrency tokens and `change_version` are also configured by convention.

Expose the set on the abstraction:

```csharp
public interface IAppDbContext
{
    DbSet<ExpenseCategory> Categories { get; }
    // …
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

### Step 3: Migrations for every Tier 1/2 provider

```bash
for p in PostgreSql SqlServer MySql Oracle; do
  dotnet ef migrations add AddExpenseCategories \
    -p src/Backend/Outlay.Persistence.$p -s src/Backend/Outlay.Api -- --Database:Provider=$p
done
```

Review each generated migration (names, lengths, Unicode). CI runs `dotnet ef migrations has-pending-model-changes` for each provider.

### Step 4: Command, validator, handler

```csharp
// Application/Features/Categories/CreateCategory/CreateCategoryCommand.cs
public sealed record CreateCategoryCommand(string Name, string? Description)
    : ICommand<Result<CategoryDto>>, IRequirePermission, ITransactionalCommand, IIdempotentCommand, IInvalidatesCache
{
    public string Permission => Permissions.Categories.Manage;
    public IReadOnlyList<string> TagsToInvalidate => [CacheTags.Categories];   // tenant prefix added by IAppCache
}
```

```csharp
// CreateCategoryValidator.cs: input shape only; uniqueness is checked in the handler
public sealed class CreateCategoryValidator : AbstractValidator<CreateCategoryCommand>
{
    public CreateCategoryValidator(IStringLocalizer<SharedResource> l)
    {
        RuleFor(x => x.Name).NotEmpty().WithMessage(l["Category.NameRequired"])
                            .MaximumLength(ExpenseCategory.NameMaxLength).WithMessage(l["Category.NameTooLong"]);
        RuleFor(x => x.Description).MaximumLength(250);
    }
}
```

```csharp
// CreateCategoryHandler.cs
internal sealed class CreateCategoryHandler(IAppDbContext db, ITenantContext tenant, IPlanLimits limits)
    : ICommandHandler<CreateCategoryCommand, Result<CategoryDto>>
{
    public async Task<Result<CategoryDto>> Handle(CreateCategoryCommand cmd, CancellationToken ct)
    {
        var created = ExpenseCategory.Create(tenant.TenantId, cmd.Name, cmd.Description);
        if (created.IsFailure) return created.Error;
        var category = created.Value;

        if (await db.Categories.TagWithCaller().AnyAsync(c => c.NormalizedName == category.NormalizedName, ct))
            return CategoryErrors.DuplicateName;                                   // BR-CAT-01 → 409

        var limit = await limits.EnsureCanAddCategoryAsync(ct);                    // BR-CAT-05 / BR-PLAN-02
        if (limit.IsFailure) return limit.Error;

        db.Categories.Add(category);
        return category.ToDto();             // SaveChanges + commit happen in TransactionBehavior
    }
}
```

Mapping is explicit (ADR-0016):

```csharp
// Application/Features/Categories/CategoryMappings.cs
internal static class CategoryMappings
{
    public static CategoryDto ToDto(this ExpenseCategory c) => new(c.Id, c.Name, c.Description, c.IsActive);

    // Expression form, translatable to SQL, for queries
    public static readonly Expression<Func<ExpenseCategory, CategoryDto>> Projection =
        c => new CategoryDto(c.Id, c.Name, c.Description, c.IsActive);
}

public sealed record CategoryDto(Guid Id, string Name, string? Description, bool IsActive);
```

> **Why a duplicate check in the handler *and* a unique index?** The handler gives a friendly 409 in the normal case. The index guarantees correctness under concurrency, and `ExceptionAdvisor` maps the unique-violation exception to 409 if two requests race.

### Step 5: Queries (get + advanced search)

```csharp
public sealed record GetCategoryQuery(Guid Id) : IQuery<Result<CategoryDto>>, IRequirePermission, ICacheableQuery
{
    public string Permission => Permissions.Categories.Read;
    public string CacheKey => $"categories:{Id}";
    public IReadOnlyList<string> Tags => [CacheTags.Categories];
}

internal sealed class GetCategoryHandler(IAppDbContext db, IDatabaseCapabilities caps)
    : IQueryHandler<GetCategoryQuery, Result<CategoryDto>>
{
    public async Task<Result<CategoryDto>> Handle(GetCategoryQuery q, CancellationToken ct)
        => await db.Categories.AsNoTracking().TagWithCaller()
               .Where(c => c.Id == q.Id)
               .ProjectSafe(caps, CategoryMappings.Projection)       // server Select, or in-memory on MongoDB
               .FirstOrDefaultAsync(ct) is { } dto ? dto : CategoryErrors.NotFound;
}
```

Register searchable fields **once**. The generic search slice, the `/search/fields` metadata endpoint and the FilterBuilder UI all use this map:

```csharp
// Application/Features/Categories/SearchCategories/CategorySearchFields.cs
internal sealed class CategorySearchFields : SearchFieldMap<ExpenseCategory>
{
    public CategorySearchFields()
    {
        Text("name",        c => c.NormalizedName, labelKey: "Category.Field.Name", normalize: true);
        Text("description", c => c.Description,    labelKey: "Category.Field.Description");
        Bool("isActive",    c => c.IsActive,       labelKey: "Category.Field.Status");
        DefaultSort("name");
    }
}

public sealed record SearchCategoriesQuery(SearchRequest Request)
    : IQuery<Result<PagedResult<CategoryDto>>>, IRequirePermission
{
    public string Permission => Permissions.Categories.Read;
}

internal sealed class SearchCategoriesHandler(IAppDbContext db, ISearchEngine search, IDatabaseCapabilities caps)
    : IQueryHandler<SearchCategoriesQuery, Result<PagedResult<CategoryDto>>>
{
    public Task<Result<PagedResult<CategoryDto>>> Handle(SearchCategoriesQuery q, CancellationToken ct)
        => search.RunAsync<ExpenseCategory, CategoryDto>(
               db.Categories.AsNoTracking().TagWithCaller(), q.Request, CategoryMappings.Projection, caps, ct);
}
```

`ISearchEngine` validates fields and operators against the map, builds the expression tree (JSON filter or OQL), applies sorting and paging, and returns `400` with a pointer to the bad clause if anything is unknown. The tenant filter is already applied by EF. For expenses you'd also call `IDataScopeService.ApplyScope(...)` first.

### Step 6: Endpoints

```csharp
// src/Backend/Outlay.Api/Endpoints/Categories/CreateCategoryEndpoint.cs
internal sealed class CreateCategoryEndpoint : IEndpoint
{
    public void Map(IEndpointRouteBuilder app) => app.MapGroup("/api/v1/categories").WithTags("Categories")
        .MapPost("/", async (CreateCategoryCommand cmd, ISender sender, CancellationToken ct) =>
            (await sender.Send(cmd, ct)).ToCreatedResult(dto => $"/api/v1/categories/{dto.Id}"))
        .AddEndpointFilter<IdempotencyFilter>()
        .RequirePermission(Permissions.Categories.Manage)
        .WithName("CreateCategory")
        .WithSummary("Create an expense category")
        .WithDescription("Creates a category in the current organisation. Names are unique (case-insensitive), " +
                         "max 50 characters. Starter plan: max 10 categories (403 plan_limit_reached).")
        .Produces<CategoryDto>(StatusCodes.Status201Created)
        .ProducesValidationProblem()
        .ProducesProblem(StatusCodes.Status409Conflict)
        .ProducesProblem(StatusCodes.Status403Forbidden);
}
```

The search endpoint is one line per resource thanks to a helper:

```csharp
app.MapSearchEndpoints<ExpenseCategory, CategoryDto, SearchCategoriesQuery>("/api/v1/categories", "Categories",
    permission: Permissions.Categories.Read);       // adds POST /search and GET /search/fields with full OpenAPI docs
```

Endpoints are discovered automatically (`app.MapEndpoints()` scans `IEndpoint` types). Don't register them by hand.

### Step 7: Permissions

Add constants (they're exported to TypeScript at build time):

```csharp
public static partial class Permissions
{
    public static class Categories
    {
        public const string Read   = "categories.read";
        public const string Manage = "categories.manage";
    }
}
```

Add them to the role templates in `RoleTemplates.cs` (Admin: both, Manager/Member: read), then regenerate the frontend file: `npm run gen:permissions` in `src/Frontend/outlay-web`.

### Step 8: Bulk import

```csharp
internal sealed class CategoryImportDefinition(IAppDbContext db)
    : IImportDefinition<CategoryImportRow, CreateCategoryCommand>
{
    public string Resource => "categories";
    public IReadOnlyList<ImportColumn> Columns =>
    [
        new("Name",        bnHeader: "নাম",   required: true,  type: ImportType.Text),
        new("Description", bnHeader: "বিবরণ", required: false, type: ImportType.Text),
    ];

    public CreateCategoryCommand ToCommand(CategoryImportRow row) => new(row.Name, row.Description);   // reuses the validator
    public IEnumerable<string> DuplicateKey(CategoryImportRow row) => [CategoryName.Create(row.Name).Value?.Normalized ?? ""];

    public async Task<ISet<string>> ExistingKeysAsync(IEnumerable<CategoryImportRow> rows, CancellationToken ct)
    {
        var keys = rows.Select(r => CategoryName.Create(r.Name).Value?.Normalized).OfType<string>().Distinct().ToList();
        return (await db.Categories.TagWithCaller().Where(c => keys.Contains(c.NormalizedName))
                                   .Select(c => c.NormalizedName).ToListAsync(ct)).ToHashSet();
    }
}
```

Registering the definition gives you `GET /api/v1/categories/import/template`, `POST /api/v1/categories/import` and the error-file download, with all-or-nothing behaviour (ADR-0015). Plan limits are checked across the whole file (an 11th category on Starter fails the import).

### Step 9: Integration tests

```csharp
// tests/Backend/Outlay.IntegrationTests/Categories/CreateCategoryTests.cs
// xUnit v3 + AwesomeAssertions. ApiFactory = WebApplicationFactory<Program> with Testcontainers (PostgreSQL + Redis)
[Collection(nameof(ApiCollection))]
public sealed class CreateCategoryTests(ApiFactory api) : IAsyncLifetime
{
    public async ValueTask InitializeAsync() => await api.ResetDatabaseAsync();   // Respawn
    public ValueTask DisposeAsync() => ValueTask.CompletedTask;

    [Fact]
    public async Task Admin_creates_category_201()
    {
        var client = await api.ClientFor(TestUsers.TechCorpAdmin);
        var res = await client.PostAsJsonAsync("/api/v1/categories", new { name = "Travel" }, WithIdempotencyKey());
        res.StatusCode.Should().Be(HttpStatusCode.Created);
    }

    [Fact]
    public async Task Duplicate_name_case_insensitive_409()
    {
        var client = await api.ClientFor(TestUsers.TechCorpAdmin);
        await client.PostAsJsonAsync("/api/v1/categories", new { name = "Travel" }, WithIdempotencyKey());
        var res = await client.PostAsJsonAsync("/api/v1/categories", new { name = "  TRAVEL " }, WithIdempotencyKey());
        (await res.ReadProblemAsync()).Code.Should().Be("category.duplicate_name");
    }

    [Fact]
    public async Task Member_cannot_create_403()
        => (await (await api.ClientFor(TestUsers.TechCorpMember))
              .PostAsJsonAsync("/api/v1/categories", new { name = "X" }, WithIdempotencyKey()))
           .StatusCode.Should().Be(HttpStatusCode.Forbidden);

    [Fact]
    public async Task Other_tenant_cannot_read_404()                // tenant isolation
    {
        var id = await api.SeedCategoryAsync(TestTenants.TechCorp, "Travel");
        var res = await (await api.ClientFor(TestUsers.MedicoreAdmin)).GetAsync($"/api/v1/categories/{id}");
        res.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task Same_idempotency_key_replays_response()
    {
        var client = await api.ClientFor(TestUsers.TechCorpAdmin);
        var key = Guid.CreateVersion7().ToString();
        var a = await client.PostAsJsonAsync("/api/v1/categories", new { name = "Meals" }, WithIdempotencyKey(key));
        var b = await client.PostAsJsonAsync("/api/v1/categories", new { name = "Meals" }, WithIdempotencyKey(key));
        b.Headers.GetValues("Idempotent-Replayed").Single().Should().Be("true");
        (await b.Content.ReadAsStringAsync()).Should().Be(await a.Content.ReadAsStringAsync());
    }
}
```

### Step 10: Postman

Add the requests to `postman/Outlay.postman_collection.json` under `02 Admin` (create, update, deactivate, search, import) and `04 User (Member)` (read), plus rows in `05 RBAC negative matrix`. The collection-level scripts already handle tokens, correlation IDs and idempotency keys (TRD §14.4).

### Step 11: Docs & release notes

- Update the spec sections you touched (PRD rule IDs, Schema table).
- Add an entry to `RELEASENOTES.md → [Unreleased]` with SQA checklist rows, for example: *"Categories · As Admin create 'Travel' then ' travel ' · second request returns 'This category already exists' · Admin"*.
- Update `AI-HANDOVER.md` and tick the Plan task.

---

## 4. Cross-cutting how-tos

### 4.1 Logging

```csharp
logger.AppInfo("Budget saved for {Year} with {CategoryCount} categories", year, count);   // caller info added automatically
logger.AppWarning("Import {JobId} took {ElapsedMs} ms", jobId, elapsed);
logger.AppError(ex, "Notification delivery failed for {OutboxId}", id);                   // → error + exception logs
```

- Use message templates (`{Name}`), **never** string interpolation.
- **Never log** money amounts, passwords, OTPs, tokens or raw emails/phones. Log ids and counts instead (`{ExpenseId}`, not `{Amount}`). Mark DTO properties `[Sensitive]` so destructuring masks them.
- Audit-worthy actions: `auditLogger.Log(AuditAction.PermissionGranted, entity: membershipId, details: new { permissions })`.

### 4.2 Tagging queries for the query log

Always call `.TagWithCaller()` on an EF query (or use `IDapperExecutor`). The query log then shows the class, method, file and line, and slow queries point straight at your code. Architecture tests warn when a handler queries `IAppDbContext` without it.

### 4.3 Returning errors vs throwing

| Situation | Do |
|---|---|
| Business rule fails (duplicate, over limit, not found, forbidden) | `return SomeErrors.X;` (`Result`) |
| Input shape invalid | FluentValidation rule (runs in `ValidationBehavior`) |
| Programming error / impossible state | `throw` (becomes 500 + exception log) |
| A dependency is down | Let it throw (`GlobalExceptionMiddleware` maps it to 503 with the support contact) |

### 4.4 Teaching ExceptionAdvisor a new failure

```csharp
// Infrastructure/Logging/Advice/ImportAdvice.cs
internal sealed class ImportFileLockedAdvice : IExceptionAdviceRule
{
    public bool Matches(Exception ex) => ex is IOException io && io.HResult == unchecked((int)0x80070020);
    public ExceptionAdvice Advice => new(
        ProbableCauseKey: "Advice.FileLocked.Cause",                  // "The uploaded file is open in another process."
        FixKeys: ["Advice.FileLocked.Fix1", "Advice.FileLocked.Fix2"]);
}
```

Rules are discovered by DI scanning. Their text lives in `Resources/en.json` / `bn.json`.

### 4.5 Caching

- Make a query cacheable with `ICacheableQuery` (key + tags). Invalidate from commands with `IInvalidatesCache`. Invalidation runs **after** commit.
- Never build keys by hand. `IAppCache`/`CacheKeys` add the `t:{tenantId}:` prefix.
- Don't cache anything user-specific under a tenant-wide key. Include the user id or data scope in the key (`expenses:list:{scope}:{userId}:{hash}`).

### 4.6 Idempotency

Mark the command `IIdempotentCommand` and add `.AddEndpointFilter<IdempotencyFilter>()` to the endpoint. The filter stores and replays responses (TRD §11.3). Don't implement your own duplicate-submission checks.

### 4.7 Domain events & side effects

```csharp
// In the entity
public Result Approve(MembershipId approver, string? note, DateTimeOffset now)
{
    if (Status != ExpenseStatus.PendingApproval) return ExpenseErrors.NotPending;
    if (approver == MembershipId) return ExpenseErrors.CannotApproveOwn;
    (Status, ApprovedBy, ApprovedAt, ApprovalNote) = (ExpenseStatus.Approved, approver, now, note);
    Raise(new ExpenseApproved(Id, TenantId, CategoryId, MembershipId, ExpenseDate, Amount));
    return Result.Success();
}

// A handler in Application: runs inside the same transaction, before commit
internal sealed class UpdateDailyTotalsOnApproved(IAppDbContext db) : IDomainEventHandler<ExpenseApproved> { … }
```

- **In-transaction** handlers: totals, alert events, notification rows, outbox rows.
- **After-commit** work (email/SMS, SignalR push) goes through the **outbox** or `IAfterCommit`, never directly from a handler.

### 4.8 Background jobs

```csharp
await queue.EnqueueAsync(new ImportJobRequested(jobId), ct);          // IBackgroundQueue (bounded Channel)

internal sealed class ImportWorker(IBackgroundQueue queue, IServiceScopeFactory scopes, ILogger<ImportWorker> log)
    : BackgroundService { /* create a scope per job, set the tenant context from the job, run the pipeline */ }
```

Scheduled jobs must take `IDistributedLock` (`using var l = await locks.TryAcquireAsync("jobs:monthly-snapshot", TimeSpan.FromMinutes(10), ct); if (l is null) return;`) so only one replica runs them.

### 4.9 Localisation

- Add the key to **both** `Resources/en.json` and `Resources/bn.json` (`Category.DuplicateName`: "This category already exists." / "এই খাত আগে থেকেই আছে।").
- Inject `IStringLocalizer<SharedResource>`. Validators and ProblemDetails use it automatically. A unit test fails when the two files' keys differ.
- Numbers in server-generated text: `amount.ToString("N2", culture)`, where the culture comes from the request.

### 4.10 Permissions & data scope

```csharp
// Command/query: declare it
public string Permission => Permissions.Expenses.Approve;

// Expense/report queries: ALWAYS scope them
var query = scope.ApplyScope(db.Expenses.AsNoTracking().TagWithCaller());   // own / team / all
```

Need "any of" permissions? Implement `IRequireAnyPermission`. Don't check `ICurrentUser.Role` in handlers. Roles are only templates.

### 4.11 Multi-tenancy rules

- Tenant id **only** from `ITenantContext` (token). Never accept `tenantId` in a request body.
- `IgnoreQueryFilters(["Tenant"])` is allowed only in `Features/Tenants`, `Features/System`, `Features/Reports/Platform` (architecture test).
- Every new endpoint gets a cross-tenant test (Step 9).

### 4.12 Adding an assistant intent

1. Add an `IntentDefinition` in `Application/Assistant/Intents/` with EN and BN patterns, slots, the required permission and the existing query to call.
2. Add answer templates to `en.json`/`bn.json`.
3. Add tests: exact-phrase matches, Bengali phrasing, a refusal for a role without permission, and an out-of-scope person ("I couldn't find that person in your team").
4. Never fetch data directly in the intent. Always send an existing query through `ISender` so `AuthorizationBehavior` applies.

---

## 5. Testing guide

| Project | Use it for | Style |
|---|---|---|
| `Outlay.UnitTests` → `Domain/` | Entities, value objects, `BudgetMonitor`, plan rules | `[Theory]` + `[InlineData]` tables. No test doubles. |
| `Outlay.UnitTests` → `Application/` | Handlers, validators, behaviours, filter compiler, import pipeline, assistant | Substitute only `Application/Abstractions` with NSubstitute. Use `FakeClock`, the in-memory `IAppCache` fake and `TestDbContext` (SQLite in-memory) when you need LINQ. |
| `Outlay.UnitTests` → `Infrastructure/` | BCrypt hasher, OTP, masking/redaction, log file naming, ExceptionAdvisor rules | |
| `Outlay.ArchitectureTests` | Layer rules, no cross-slice references, naming, endpoint documentation, tenant-filter usage | Fails the build |
| `Outlay.IntegrationTests` | Real HTTP + real DB (Testcontainers), per provider. Includes "log file written to `logs/<category>/dd-MM-yyyy.txt`" checks. | One class per endpoint group. Respawn resets data. |
| `Outlay.PerformanceTests` | Hot paths (filter building, import validation, report aggregation) | `dotnet run -c Release --project tests/Backend/Outlay.PerformanceTests` |
| `Outlay.LoadTests` | k6 target-RPS scripts | `k6 run --env BASE_URL=https://staging… tests/Backend/Outlay.LoadTests/load.js` |
| `Outlay.StressTests` | k6 stress, spike, soak + failure injection | `tests/Backend/Outlay.StressTests/run-failure-drill.sh` runs k6, then `docker compose stop redis` (expect 0 errors) and `stop postgres` (expect 503 + support contact, then recovery) |

All .NET test projects use **xUnit + NSubstitute + AwesomeAssertions** (ADR-0017). Run only domain tests with `dotnet test tests/Backend/Outlay.UnitTests --filter "FullyQualifiedName~Outlay.UnitTests.Domain"`.

**Handler unit test example (xUnit + NSubstitute + AwesomeAssertions):**

```csharp
public sealed class ApproveExpenseHandlerTests
{
    private readonly ICurrentUser _user = Substitute.For<ICurrentUser>();
    private readonly FakeClock _clock = new(new DateTimeOffset(2026, 6, 15, 9, 0, 0, TimeSpan.FromHours(6)));

    [Fact]
    public async Task Manager_cannot_approve_own_expense()
    {
        await using var db = TestDbContext.Create();
        var expense = ExpenseBuilder.Pending().OwnedBy(MembershipIds.Nusrat).Build();
        db.Expenses.Add(expense); await db.SaveChangesAsync();
        _user.MembershipId.Returns(MembershipIds.Nusrat);

        var result = await new ApproveExpenseHandler(db, _user, _clock)
            .Handle(new ApproveExpenseCommand(expense.Id, null), TestContext.Current.CancellationToken);

        result.Error.Should().Be(ExpenseErrors.CannotApproveOwn);
    }

    [Fact]
    public async Task Approving_raises_ExpenseApproved_once()
    {
        await using var db = TestDbContext.Create();
        var expense = ExpenseBuilder.Pending().OwnedBy(MembershipIds.Fahmida).Build();
        db.Expenses.Add(expense); await db.SaveChangesAsync();
        _user.MembershipId.Returns(MembershipIds.Nusrat);

        await new ApproveExpenseHandler(db, _user, _clock)
            .Handle(new ApproveExpenseCommand(expense.Id, "OK"), TestContext.Current.CancellationToken);

        expense.DomainEvents.OfType<ExpenseApproved>().Should().ContainSingle();
    }
}
```

`TestContext.Current.CancellationToken` is xUnit v3's per-test token, so cancelled test runs stop promptly.

Rules:
- A bug fix starts with a failing test.
- No `Thread.Sleep` and no real time. Use `FakeClock`.
- Integration tests use the **seeded personas** (`TestUsers.TechCorpAdmin`, `TestUsers.MedicoreAdmin`…), so RBAC and isolation tests read like the requirement.

---

## 6. Databases, migrations and providers

### 6.1 Migrations workflow

1. Change the entity/configuration.
2. Add the migration for **PostgreSql, SqlServer, MySql, Oracle** (loop in §3 Step 3).
3. Run the integration tests on PostgreSQL and SQL Server locally (`dotnet test tests/Backend/Outlay.IntegrationTests --filter Provider=SqlServer`).
4. MongoDB, Access and InMemory need no migration. If you added an index, update `MongoIndexInitializer`.
5. Production: the `deploy.yml` workflow runs `Outlay.Api --migrate` as a one-off job **before** the rolling update. `ApplyMigrationsOnStartup` is for Development only.

**Migrations are source code, so never ignore them in git.** Both the Development startup path and the production migration job need them. The repo's `.gitignore` already leaves them tracked. Only its explanatory comment needs updating:

```gitignore
# Migrations/                → EF Core migrations are source code (one folder per provider in
#                              src/Backend/Outlay.Persistence.<Provider>/Migrations). Applied by
#                              `Outlay.Api --migrate` before each deployment; on app startup only in
#                              Development (Database:ApplyMigrationsOnStartup=true).
```

### 6.2 Writing a Dapper read store (relational providers only)

```csharp
internal sealed class SqlReportReadStore(IDapperExecutor dapper, ITenantContext tenant) : IReportReadStore
{
    public Task<IReadOnlyList<CategorySpendRow>> GetMonthlyCategorySpendAsync(YearMonth month, CancellationToken ct)
        => dapper.QueryAsync<CategorySpendRow>(
               """
               SELECT category_id AS CategoryId, SUM(amount) AS Spent, SUM(entry_count) AS Entries
               FROM daily_category_totals
               WHERE tenant_id = @TenantId AND day >= @From AND day < @To
               GROUP BY category_id
               """,
               new { tenant.TenantId, From = month.FirstDay, To = month.NextMonth.FirstDay }, ct);
               // IDapperExecutor adds caller info + timing to the query log
}
```

- **Always** filter by `tenant_id` yourself. Dapper doesn't see EF query filters.
- Use parameters only. Never concatenate values into SQL.
- Keep SQL portable (ANSI). Put provider-specific variants behind `IDatabaseCapabilities.Dialect` only when they're unavoidable.

### 6.3 Adding a new provider

1. Create `src/Backend/Outlay.Persistence.<Name>` referencing Infrastructure.
2. Implement `IDatabaseProvider` (name, capabilities, `ConfigureDbContext`, `CreateDapperConnection`, `BackupStrategy`).
3. Register it in `ProviderRegistry` behind an MSBuild switch (`IncludeXProvider`).
4. Add the Testcontainers fixture and a nightly CI job, then document its tier in TRD §6.2.

---

## 7. Debugging playbook

| Symptom | Where to look |
|---|---|
| User reports "Something went wrong, reference 8f3c21a0b7e4" | `GET /api/v1/system/logs/exception/<dd-MM-yyyy>` (or the file), search for the **correlation ID**. The entry shows class, file, method, line, probable cause and suggested fixes. |
| A screen is slow | `application-query-logs/<date>.txt`. Search `[SLOW QUERY`, then follow the `Caller:` line to the handler. Check the index from the hint. |
| 403 you didn't expect | Audit log for the user. `GET /api/v1/me/permissions` with their token. Check that `pver` in the token is current (the token may be stale for up to 60 s after a change). |
| Data "missing" for one tenant | Tenant filter working as designed? Confirm the token's `tenant_id` claim (`/connect/introspect` in dev). |
| Duplicate records | Was an `Idempotency-Key` sent? Look in `idempotency_records`. Offline clients always send one. |
| Redis outage | Warnings "Distributed cache unavailable, serving from memory". The app keeps working, so no action is needed beyond restoring Redis. |
| OTP never arrives | `Sms:Provider`/`Email` config, outbox rows with `status=Failed` and `last_error`, Mailpit in dev |

Local tips: run with `OutlayLogging__Categories__Query__LogAllQueries=true`, and use the `.http` files in `src/Backend/Outlay.Api/Http/` (one per feature) for quick manual calls.

---

## 8. Upgrading .NET and packages

**Moving to .NET 11 (or later):**

1. Check the CI "next framework" job. It has been building against the next .NET for months, so most breaks are already known.
2. Change `<OutlayTargetFramework>net11.0</OutlayTargetFramework>` in `Directory.Build.props` and the SDK in `global.json`.
3. Bump Microsoft packages in `Directory.Packages.props` (ASP.NET Core, EF Core, extensions) to the matching major.
4. Check each EF provider has a release for the new EF major. Disable any laggard with its MSBuild switch (`-p:IncludeMySqlProvider=false`) and record that in AI-HANDOVER.
5. `dotnet build -warnaserror` and fix obsoletion warnings. Run the full test matrix including Tier 2 providers. Run a k6 load test against staging.
6. Update TRD §3/§18, ROADMAP and RELEASENOTES (with an SQA regression checklist).

**Routine package updates:** Dependabot/Renovate PRs. Patch updates merge when CI is green. Minor and major updates need a human review of the changelog, especially MediatR, OpenIddict and EF providers.

---

## 9. Pull-request checklist

- [ ] Slice follows the folder layout. The endpoint has no logic.
- [ ] Domain rules live in entities/value objects, with xUnit `[Theory]` tests.
- [ ] Validator present, with localised messages in EN and BN.
- [ ] `IRequirePermission` declared. Expense/report queries use `IDataScopeService`.
- [ ] No tenant id taken from input. Cross-tenant test added.
- [ ] Commands that create are `IIdempotentCommand` + filter. Caching tags are declared.
- [ ] `TagWithCaller()` on every EF query. No sensitive values logged.
- [ ] OpenAPI summary, description, tags, `Produces*`. Scalar shows it correctly.
- [ ] Migrations for PostgreSql, SqlServer, MySql, Oracle.
- [ ] Integration tests: success, 422, 403 per role, 404 cross-tenant, idempotent replay.
- [ ] Postman requests (role folders + RBAC matrix).
- [ ] RELEASENOTES `[Unreleased]` + SQA rows. AI-HANDOVER updated. Plan task ticked.

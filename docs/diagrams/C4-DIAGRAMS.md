# Outlay — C4 Architecture Diagrams

| Field | Value |
|---|---|
| Version | 0.0.0 · 2026-10-28 |
| Model | [C4 model](https://c4model.com/): System Context → Containers → Components → Code, plus Dynamic and Deployment views |
| Format | Mermaid (renders on GitHub/GitLab and in VS Code with a Mermaid extension). Exported SVGs go in `docs/diagrams/svg/` at milestone M13-08. |
| Related | [TRD.md](../TRD.md) (technical detail) · [Schema.md](../Schema.md) (data) · [AppFlow.md](../AppFlow.md) (user flows) |

> **This file is the canonical home of the architecture diagrams.** TRD §2 points here. When the architecture changes, update the diagram in the same PR as the code.

**How to read them:** boxes are people, software systems, containers (separately deployable or runnable things) or components (groupings inside a container). Arrows show the main direction of a dependency or data flow, labelled with purpose and protocol.

**Notation:** standard C4 colours: dark blue = person, blue = Outlay system, mid blue = container, light blue = component, grey = external system, dashed box = boundary. Levels 1–3 are drawn as Mermaid flowcharts in C4 notation, because Mermaid's experimental `C4*` diagram types lay larger views out in a single column with crossing arrows. The deployment view uses Mermaid's native `C4Deployment`, and the dynamic view uses sequence notation.

---

## Level 1 — System Context

*Who uses Outlay and which outside systems it depends on.*

```mermaid
flowchart TB
    member["<b>Member</b><br/>[Person]<br/>Records own expenses"]
    manager["<b>Manager</b><br/>[Person]<br/>Reviews team spending, approves over-budget entries"]
    admin["<b>Admin</b><br/>[Person]<br/>Manages team, categories, budgets, permissions"]
    sa["<b>Super Admin</b><br/>[Person]<br/>Operates the platform, never sees expense data"]
    sqa["<b>SQA Engineer</b><br/>[Person]<br/>Verifies releases via API and Postman"]

    outlay["<b>Outlay</b><br/>[Software System]<br/>Multi-tenant expense tracking and budget control.<br/>Installable PWA plus REST API"]

    sms["<b>SMS gateway</b><br/>[External System]<br/>Twilio (prod) or TextBelt (testing)"]
    smtp["<b>Email service</b><br/>[External System]<br/>SMTP relay"]
    speech["<b>Browser speech services</b><br/>[External System]<br/>Web Speech API"]

    member -- "Adds and views own expenses<br/>[HTTPS]" --> outlay
    manager -- "Reviews, approves, reports<br/>[HTTPS]" --> outlay
    admin -- "Administers organisation<br/>[HTTPS]" --> outlay
    sa -- "Operates platform, logs, backups<br/>[HTTPS]" --> outlay
    sqa -- "Reads release notes, runs API suites<br/>[HTTPS]" --> outlay
    outlay -- "Sends OTP and alerts<br/>[HTTPS]" --> sms
    outlay -- "Sends OTP, invitations, alerts<br/>[SMTP/TLS]" --> smtp
    outlay -. "Voice input/output in the user's browser" .-> speech

    class member,manager,admin,sa,sqa person
    class outlay system
    class sms,smtp,speech external
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#0b2545
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff
    classDef boundary fill:none,stroke:#888888,stroke-dasharray:6 4,color:#444444
```

| Element | Responsibility | Trust boundary |
|---|---|---|
| Member / Manager / Admin | Tenant users. One login may belong to several organisations with different roles. | Authenticated, tenant-scoped |
| Super Admin | Platform operations only | Authenticated, platform scope, step-up OTP for sensitive actions |
| SMS gateway / Email | Delivery of OTP and alerts | External. Credentials in secrets. Calls wrapped in retry + circuit breaker. |
| Browser speech | Speech-to-text / text-to-speech on the user's device | External to Outlay. The user sees a privacy note. |

---

## Level 2 — Containers

*The deployable and runnable pieces of Outlay and how they talk.*

```mermaid
flowchart TB
    user["<b>Any user</b><br/>[Person]<br/>Member, Manager, Admin, Super Admin"]

    subgraph outlay["Outlay [Software System]"]
        direction TB
        subgraph browser["User's browser"]
            direction LR
            spa["<b>Outlay Web (PWA)</b><br/>[Container: React 19, TypeScript, Tailwind v4]<br/>UI in English and Bengali, service worker,<br/>install prompt, offline outbox sync"]
            idb[("<b>IndexedDB</b><br/>[Container: Dexie]<br/>Cached data, outbox,<br/>sync cursor, support contact")]
        end
        lb["<b>Load balancer</b><br/>[Container: nginx]<br/>TLS, least-conn, WebSockets, edge rate limit,<br/>forwarded headers. Routes: / → SPA files;<br/>/api, /connect, /hubs, /scalar → API"]
        api["<b>Outlay API</b><br/>[Container: .NET 10, ASP.NET Core, OpenIddict, MediatR, EF Core]<br/>REST API, OAuth 2.0/OIDC server, SignalR hub,<br/>background workers. N stateless replicas"]
        subgraph data["Data stores"]
            direction LR
            db[("<b>Primary database</b><br/>[Container: configurable provider]<br/>PostgreSQL, SQL Server, MySQL, Oracle,<br/>MongoDB, MS Access, InMemory")]
            redis[("<b>Redis</b><br/>[Container: Redis 7+]<br/>L2 cache, backplanes, locks,<br/>rate limits, Data Protection keys")]
            files[("<b>File volumes</b><br/>[Container: Docker volumes]<br/>logs/category/dd-MM-yyyy.txt,<br/>backups, import error files")]
        end
    end

    sms["<b>SMS gateway</b><br/>[External System]<br/>Twilio / TextBelt"]
    smtp["<b>Email (SMTP)</b><br/>[External System]<br/>Mailpit in development"]

    user -- "Uses<br/>[HTTPS]" --> spa
    spa -- "Reads/writes offline data" --> idb
    spa -- "API, tokens, real-time<br/>[HTTPS/WSS]" --> lb
    lb -- "/api /connect /hubs /scalar<br/>[HTTP]" --> api
    api -- "Reads/writes<br/>[EF Core, Dapper, Mongo driver]" --> db
    api -- "Cache, pub/sub, locks<br/>[RESP]" --> redis
    api -- "Writes" --> files
    api -- "Sends OTP/alerts<br/>[HTTPS]" --> sms
    api -- "Sends email<br/>[SMTP]" --> smtp

    class user person
    class spa,lb,api,idb,db,redis,files container
    class sms,smtp external
    class outlay,browser,data boundary
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#0b2545
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff
    classDef boundary fill:none,stroke:#888888,stroke-dasharray:6 4,color:#444444
```

| Container | Technology | Scales by | Key decisions |
|---|---|---|---|
| Outlay Web | React 19 PWA | CDN/nginx static files | ADR-0012 (offline), ADR-0018 (stack) |
| Load balancer | nginx | Single instance (or managed LB) | TRD §11.6 |
| Outlay API | .NET 10 | Horizontal replicas (stateless) | ADR-0002…0011 |
| Primary database | Configurable provider | Vertical, read replicas later | ADR-0008, ADR-0009 |
| Redis | Redis 7 | Single node (Sentinel/managed in production) | ADR-0010. Optional: the API degrades gracefully without it. |
| File volumes | Docker volumes | Per replica, or a central log sink | ADR-0011 |

---

## Level 3 — Components of the Outlay API

```mermaid
flowchart TB
    spa["<b>Outlay Web (PWA)</b><br/>[Container]"]

    subgraph api["Outlay API [Container]"]
        direction TB
        mw["<b>Middleware pipeline</b><br/>[Component: ASP.NET Core middleware]<br/>Forwarded IP → Correlation ID → Request log → Global exception<br/>(caller location + advice, 503 + support contact) → Locale<br/>→ Rate limit → AuthN/AuthZ → Tenant context"]
            auth["<b>Auth server</b><br/>[Component: OpenIddict + Identity]<br/>Password, OTP, rotating refresh,<br/>tenant switch, revocation, lockout, BCrypt"]
            ep["<b>Feature endpoints</b><br/>[Component: Minimal APIs]<br/>One per use case, OpenAPI metadata,<br/>Idempotency-Key filter"]
            docs["<b>OpenAPI + Scalar</b><br/>[Component: Microsoft.AspNetCore.OpenApi + Scalar]<br/>/openapi/v1.json, /scalar with Outlay theme"]
        pipe["<b>MediatR pipeline</b><br/>[Component: behaviours]<br/>Logging → Validation → Authorization →<br/>Tenant/Plan limits → Caching (queries) → Transaction (commands)"]
        slices["<b>Feature slices</b><br/>[Component: commands, queries, handlers]<br/>Expenses, Budgets, Categories, Approvals, Users,<br/>Permissions, Reports, Imports, Sync, Assistant, System"]
            engines["<b>Shared engines</b><br/>[Component]<br/>Search + OQL, import pipeline,<br/>assistant intents, budget monitor"]
            dom["<b>Domain model</b><br/>[Component]<br/>Entities, value objects,<br/>domain events, BR-* rules"]
            bg["<b>Background workers</b><br/>[Component: Channels + BackgroundService]<br/>Imports, outbox, snapshots,<br/>backups, retention"]
            pers["<b>Persistence</b><br/>[Component: EF Core + providers + Dapper + Mongo read store]<br/>Tenant filters, audit + query-log interceptors"]
            cache["<b>Cache</b><br/>[Component: FusionCache]<br/>L1 + L2 + backplane,<br/>tenant-scoped tags"]
            hub["<b>Notifications hub</b><br/>[Component: SignalR]<br/>Notifications, import progress,<br/>permission changes"]
            logs["<b>Logging</b><br/>[Component: Serilog]<br/>5 categorised daily files,<br/>masking"]
    end

    db[("<b>Primary database</b><br/>[Container]")]
    redis[("<b>Redis</b><br/>[Container]")]
    vol[("<b>File volumes</b><br/>[Container]")]

    spa -- "HTTPS / WSS" --> mw
    mw -- "/connect/*" --> auth
    mw -- "/api/v1/*" --> ep
    ep -- "ISender.Send" --> pipe
    ep -. "Metadata" .-> docs
    pipe -- "Dispatches" --> slices
    pipe -- "Read-through, invalidation" --> cache
    slices --> engines
    slices --> dom
    slices -- "Enqueues jobs" --> bg
    slices -- "IAppDbContext / read stores" --> pers
    bg -- "Progress, alerts" --> hub
    pers --> db
    cache -- "L2 + backplane" --> redis
    hub -- "Backplane" --> redis
    mw -- "Writes" --> logs
    logs --> vol

    class spa,db,redis,vol container
    class mw,auth,ep,docs,pipe,slices,engines,dom,bg,pers,cache,hub,logs component
    class api boundary
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#0b2545
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff
    classDef boundary fill:none,stroke:#888888,stroke-dasharray:6 4,color:#444444
```

| Component | Project / folder | Notes |
|---|---|---|
| Diagnostics middleware | `Outlay.Api/Middleware` | `UseOutlayDiagnostics()` (TRD §8.4) |
| Auth server | `Outlay.Infrastructure/Identity` | ADR-0005, ADR-0006 |
| Feature endpoints | `Outlay.Api/Endpoints/<Feature>` | ADR-0004 |
| MediatR pipeline | `Outlay.Application/Behaviors` | ADR-0003 |
| Feature slices | `Outlay.Application/Features/<Feature>/<UseCase>` | ADR-0002 |
| Shared engines | `Outlay.Application/{Search, Import, Assistant}` | TRD §13 |
| Domain model | `Outlay.Domain` | No dependencies |
| Persistence | `Outlay.Infrastructure/Persistence` + `Outlay.Persistence.<Provider>` | ADR-0008, ADR-0009 |
| Cache | `Outlay.Infrastructure/Caching` | ADR-0010 |
| Background workers | `Outlay.Infrastructure/Background` | Distributed lock for scheduled jobs |
| Notifications hub | `Outlay.Api/Hubs` | Groups per tenant and user |
| Logging | `Outlay.Infrastructure/Logging` | ADR-0011 |

---

## Level 3 — Outlay API by project (the dependency rule)

*The same components grouped by the .NET project they live in. Arrows show compile-time references. Infrastructure **implements** the abstractions that Application defines (dependency inversion), so Application and Domain never reference infrastructure packages. `Outlay.ArchitectureTests` enforces this.*

```mermaid
flowchart TB
    subgraph ApiP["Outlay.Api"]
        direction TB
        a1["Middleware pipeline"]
        a2["Endpoints (IEndpoint)"]
        a3["OpenIddict endpoints"]
        a4["OpenAPI + Scalar"]
        a5["SignalR hub"]
    end
    subgraph AppP["Outlay.Application"]
        direction TB
        b1["Behaviours"]
        b2["Features (vertical slices)"]
        b3["Shared engines: Search · Import · Assistant"]
        b4["Abstractions (ports):<br/>IAppDbContext, IAppCache, ISmsSender,<br/>IEmailSender, IBackgroundQueue, IClock"]
    end
    subgraph DomP["Outlay.Domain"]
        c1["Entities · Value objects ·<br/>Domain events · Business rules"]
    end
    subgraph InfP["Outlay.Infrastructure + Outlay.Persistence.*"]
        direction TB
        d1["Persistence: EF Core, Dapper,<br/>Mongo read store, interceptors"]
        d2["Caching: FusionCache, Redis/Memcached"]
        d3["Identity: Identity, OpenIddict, BCrypt, OTP"]
        d4["Messaging: SMS, Email, Outbox"]
        d5["Logging: Serilog, masking, ExceptionAdvisor"]
        d6["Background workers, Backup strategies"]
    end

    ApiP -- "references" --> AppP
    ApiP -- "references (composition root only)" --> InfP
    AppP -- "references" --> DomP
    InfP -- "references" --> AppP
    d1 -. "implements" .-> b4
    d2 -. "implements" .-> b4
    d4 -. "implements" .-> b4
    d6 -. "implements" .-> b4

    class a1,a2,a3,a4,a5,b1,b2,b3,b4,c1,d1,d2,d3,d4,d5,d6 component
    class ApiP,AppP,DomP,InfP boundary
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#0b2545
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff
    classDef boundary fill:none,stroke:#888888,stroke-dasharray:6 4,color:#444444
```

| Rule | Enforced by |
|---|---|
| Domain references nothing | `Outlay.ArchitectureTests` |
| Application references only Domain (no EF provider, Serilog or Redis packages) | `Outlay.ArchitectureTests` |
| Api uses Infrastructure only in `Program.cs`/DI extensions | `Outlay.ArchitectureTests` |
| A feature never references another feature's internals | `Outlay.ArchitectureTests` (namespace rule per `Features.<Name>`) |

---

## Level 3 — Components of Outlay Web (PWA)

```mermaid
flowchart TB
    user["<b>User</b><br/>[Person]"]

    subgraph web["Outlay Web (PWA) [Container]"]
        direction TB
        shell["<b>App shell and router</b><br/>[Component: React Router 7]<br/>Layouts, permission-guarded lazy routes"]
        features["<b>Feature modules</b><br/>[Component: React components + hooks]<br/>Dashboard, Expenses, Approvals, Budgets, Categories,<br/>Reports, Team, Permissions, Imports, Assistant, System"]
            ds["<b>Design system</b><br/>[Component: Tailwind v4 + Radix]<br/>BudgetThread, DataTable,<br/>FilterBuilder, ImportWizard"]
            i18n["<b>Localisation</b><br/>[Component: i18next + Intl]<br/>EN/BN text, digits,<br/>dates, money, titles"]
            voice["<b>Speech adapters</b><br/>[Component: Web Speech API]<br/>Voice in, spoken answers"]
        data["<b>Data layer</b><br/>[Component: TanStack Query + http.ts]<br/>Typed client, tenant+language keys,<br/>ProblemDetails, idempotency keys"]
            authc["<b>Auth and permissions</b><br/>[Component]<br/>In-memory token, silent refresh,<br/>OTP flow, usePermission"]
            offline["<b>Offline engine</b><br/>[Component: Dexie + outbox]<br/>IndexedDB cache, queued mutations,<br/>delta pull, conflicts"]
            rt["<b>Real-time client</b><br/>[Component: SignalR]<br/>Notifications, import progress"]
            sw["<b>Service worker</b><br/>[Component: Workbox]<br/>service-worker.js: precache,<br/>background sync, updates"]
    end

    apiC["<b>Outlay API</b><br/>[Container]"]

    user -- "Navigates" --> shell
    shell --> features
    features --> ds
    features -- "t(), useFormat()" --> i18n
    features -- "Assistant panel" --> voice
    features -- "Queries, mutations" --> data
    features -- "useHubEvent()" --> rt
    data -- "Gets token" --> authc
    data -- "Falls back / queues" --> offline
    sw -- "Background sync" --> offline
    data -- "REST [HTTPS]" --> apiC
    authc -- "/connect/token" --> apiC
    rt -- "/hubs/notifications [WSS]" --> apiC

    class user person
    class apiC container
    class shell,features,ds,i18n,voice,data,authc,offline,rt,sw component
    class web boundary
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#0b2545
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff
    classDef boundary fill:none,stroke:#888888,stroke-dasharray:6 4,color:#444444
```

---

## Dynamic view — Member submits an expense over a Hard limit

*Runtime collaboration for the most important business flow (AppFlow F-03 → F-04). Drawn in sequence notation, which C4 allows for dynamic views and which stays readable with many steps.*

```mermaid
sequenceDiagram
    autonumber
    actor M as Member (Fahmida)
    participant W as Outlay Web (PWA)
    participant A as Outlay API
    participant R as Redis
    participant D as Primary database
    actor N as Manager (Nusrat)
    M->>W: Enters 1,200 taka in Travel, taps Submit for approval
    W->>A: POST /api/v1/expenses (Idempotency-Key, X-Correlation-Id)
    A->>R: Idempotency check, cached permissions and budget
    A->>D: One transaction: budget check, save PendingApproval, alert event, notifications, outbox
    A->>R: Invalidate tenant report tags (backplane tells other replicas)
    A-->>W: 201 Created, status PendingApproval
    W-->>M: Shows "Waiting for approval"
    A-)N: Approval Required, pushed in real time (SignalR), plus email/SMS on Pro
```

If step 2 fails because the device is offline, the request waits in the IndexedDB outbox and is replayed later with the **same** Idempotency-Key, so the expense is created exactly once (ADR-0012).

---

## Dynamic view — Log pipeline (from code to `logs/<category>/dd-MM-yyyy.txt`)

*How one log event travels. It covers the owner's requirements: five categories, on/off from configuration, caller location, masking of sensitive values, and red slow queries (TRD §8, ADR-0011).*

```mermaid
sequenceDiagram
    autonumber
    participant S as Source (handler, middleware, EF/Dapper/Mongo interceptor)
    participant H as Caller-info helper (AppInfo/AppError, TagWithCaller)
    participant E as Enrichers (CorrelationId, Tenant, User, ClientIp, LogCategory, LogDate)
    participant M as Masking policy (password, otp, token, amount, …)
    participant F as Category filter (OutlayLogging options, hot reload)
    participant K as Map sink → File sink
    participant C as Console sink
    S->>H: Log event (+ class, file, method, line)
    H->>E: Structured event
    E->>M: Event with context properties
    M->>F: Sensitive values replaced with ***
    alt category enabled and level ≥ minimum
        F->>K: Write to logs/<category>/dd-MM-yyyy.txt
        F->>C: Write to console
        Note over C: Slow query (≥ SlowQueryThresholdMs) in ANSI red. The file line gets the [SLOW QUERY ⚠] prefix.
    else category disabled
        F-->>S: Dropped (no I/O)
    end
```

Exceptions take the same path, plus one step. `GlobalExceptionMiddleware` resolves the first `Outlay.*` stack frame (class, file, method, line, available because PDBs are embedded) and asks `ExceptionAdvisor` for the probable cause and fixes. It writes a one-line entry to `application-error-logs` and the full entry to `application-exceptions-logs`, linked by the exception id.

---

## Deployment view — Docker Compose (development, staging)

```mermaid
C4Deployment
    title Deployment: single Docker host

    Deployment_Node(host, "Docker host", "Linux, Docker Engine") {
        Deployment_Node(edge, "lb container", "nginx:stable-alpine") {
            Container(lb, "Load balancer", "nginx", "Ports 80/443, TLS, upstream least_conn")
        }
        Deployment_Node(webn, "web container", "nginx:stable-alpine") {
            Container(web, "Outlay Web", "Static PWA build", "index.html, service-worker.js, manifest.json")
        }
        Deployment_Node(apin, "api containers x N", "mcr.microsoft.com/dotnet/aspnet:10.0") {
            Container(api, "Outlay API", ".NET 10", "Stateless replicas, port 8080")
        }
        Deployment_Node(dbn, "db container", "postgres (default profile)") {
            ContainerDb(db, "Primary database", "PostgreSQL / other profile", "Volume db-data")
        }
        Deployment_Node(redisn, "redis container", "redis:7") {
            ContainerDb(redis, "Redis", "Redis", "Cache, backplanes, locks")
        }
        Deployment_Node(mailn, "mailpit container", "axllent/mailpit") {
            Container(mail, "Mailpit", "SMTP sink", "Development email viewer on 8025 (MAILPIT_UI_PORT)")
        }
    }

    Rel(lb, web, "Static files", "HTTP")
    Rel(lb, api, "API and WebSockets", "HTTP")
    Rel(api, db, "Reads/writes", "TCP")
    Rel(api, redis, "Cache and pub/sub", "TCP")
    Rel(api, mail, "Email", "SMTP")
```

| Environment | Topology | Differences |
|---|---|---|
| Local (no Docker) | `dotnet run` + `npm run dev` | InMemory DB, no Redis (`Cache:L2=None`), OTP 111111, console SMS. **Exactly one API instance** (see warning below). |
| Docker compose (dev/demo) | The diagram above, `--scale api=2` | PostgreSQL profile by default, Mailpit, OTP 111111 |
| Docker compose, `inmemory` profile | lb + web + **one** api, no db container | For quick demos only. `--scale api` is ignored by this profile. |
| Staging | Same images. Managed DB and Redis. | Real OTP. TextBelt or Twilio test credentials. Nightly k6. |
| Production | Same images behind a managed load balancer, ≥ 3 API replicas | Real OTP via Twilio. Central log sink recommended. Migration job before rollout. Manual approval gate. |

> ⚠ **InMemory means one API instance.** The InMemory provider keeps data inside one process. With two replicas behind the load balancer, each would have its own separate data, so users would see records appear and disappear between requests. The API refuses to start with `Database:Provider=InMemory` when `Deployment:ExpectedReplicas > 1`, and the compose `inmemory` profile runs a single API container.

---

## Level 4 — Code (one vertical slice)

*Code level is shown for one representative slice only. The rest follow the same shape.*

```mermaid
classDiagram
    direction LR
    class CreateExpenseEndpoint {
        +Map(IEndpointRouteBuilder app)
    }
    class CreateExpenseCommand {
        <<record>>
        +DateOnly ExpenseDate
        +Guid CategoryId
        +decimal Amount
        +string Description
        +bool ConfirmDuplicate
        +string Permission
    }
    class CreateExpenseValidator {
        +CreateExpenseValidator(IClock, IStringLocalizer)
    }
    class CreateExpenseHandler {
        +Handle(CreateExpenseCommand, CancellationToken) Result
    }
    class IAppDbContext {
        <<interface>>
        +Expenses
        +Categories
    }
    class IBudgetMonitor {
        <<interface>>
        +EvaluateAsync(categoryId, month, amount) BudgetCheck
    }
    class Expense {
        +Record(...) Expense$
        +Approve(...) Result
        +Reject(...) Result
    }
    class ExpenseDto {
        <<record>>
    }
    CreateExpenseEndpoint ..> CreateExpenseCommand : binds
    CreateExpenseEndpoint ..> ISender : sends
    CreateExpenseValidator ..> CreateExpenseCommand : validates
    CreateExpenseHandler ..> CreateExpenseCommand : handles
    CreateExpenseHandler --> IAppDbContext
    CreateExpenseHandler --> IBudgetMonitor
    CreateExpenseHandler ..> Expense : creates
    CreateExpenseHandler ..> ExpenseDto : returns
    class ISender {
        <<interface>>
    }
```

---

## Maintenance

- Diagrams are code: edit the Mermaid here, and CI renders every block with `mmdc` to catch syntax errors.
- Keep names identical to code (`Outlay.Api`, `IAppDbContext`) and to TRD section titles.
- A new container (e.g. a separate auth server or a reporting worker) needs an ADR first, then updates to Level 2, Level 3 and the Deployment view.

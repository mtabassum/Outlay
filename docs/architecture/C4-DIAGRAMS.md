# C4 Diagrams — Outlay

Mermaid; renders on GitHub. Levels: Context (1), Containers (2), Components (3), Deployment, plus Data (ER lives in `Schema.md`).

## Level 1 — System Context
```mermaid
C4Context
  title Outlay — System Context
  Person(member, "Member", "Logs own expenses")
  Person(manager, "Manager", "Approves, reviews monthly report")
  Person(admin, "Tenant Admin", "Manages team, budgets, permissions")
  Person(sa, "Super Admin", "Platform operator; no tenant expense access")
  System(outlay, "Outlay", "Multi-tenant expense tracking & budget control (PWA + API)")
  System_Ext(sms, "SMS Gateway", "Free gateway adapter / Twilio")
  System_Ext(mail, "SMTP / Email", "Invitations, OTP, alerts")
  Rel(member, outlay, "Uses", "HTTPS, offline PWA")
  Rel(manager, outlay, "Uses")
  Rel(admin, outlay, "Uses")
  Rel(sa, outlay, "Operates")
  Rel(outlay, sms, "Sends OTP/alerts")
  Rel(outlay, mail, "Sends email")
```

## Level 2 — Containers
```mermaid
C4Container
  title Outlay — Containers
  Person(user, "Users")
  System_Boundary(o, "Outlay") {
    Container(lb, "Nginx", "Reverse proxy / LB", "TLS, least-conn, edge rate limit, forwarded headers")
    Container(web, "Web App", "Next.js 16+/React 19+, Tailwind", "UI, BFF session, PWA, IndexedDB outbox")
    Container(api, "API", ".NET 10, Minimal APIs, MediatR, OpenIddict", "Business logic, auth, OpenAPI/Scalar, logging middleware")
    ContainerDb(db, "Database", "InMemory | SQL Server | MySQL | Oracle | PostgreSQL | Access | MongoDB", "Primary store (EF Core or Dapper)")
    ContainerDb(redis, "Redis", "Cache + backplane", "HybridCache L2, pub/sub, rate/ticket state")
    Container(files, "Log & Backup storage", "File system / volume", "logs/<category>/dd-MM-yyyy.txt")
    Container(worker, "Background workers", "Hosted services", "Import jobs, outbox, aggregates, log writer")
  }
  System_Ext(sms, "SMS Gateway")
  System_Ext(mail, "SMTP")
  Rel(user, lb, "HTTPS")
  Rel(lb, web, "Routes /")
  Rel(lb, api, "Routes /api /connect /scalar")
  Rel(web, api, "REST + OAuth tokens")
  Rel(api, db, "SQL / driver")
  Rel(api, redis, "Cache")
  Rel(api, files, "Writes logs")
  Rel(worker, db, "Reads/writes")
  Rel(api, sms, "OTP/alerts")
  Rel(api, mail, "Email")
```

## Level 3 — API components (feature slices + shared pipeline)
```mermaid
flowchart TB
  subgraph Api[Outlay.Api]
    MW[Middleware: Forwarded IP · Correlation · Logging · RateLimit · Auth · Tenant · Locale]
    EP[Endpoint groups per feature]
    OID[OpenIddict server/validation]
    DOCS[Scalar + OpenAPI]
  end
  subgraph App[Outlay.Application]
    BEH[Behaviors: Log · Validate · Authorize · Idempotency · Cache · Transaction]
    subgraph Slices[Features / vertical slices]
      S1[Expenses]; S2[Budgets]; S3[Categories]; S4[Approvals]; S5[Team & Invitations]; S6[Permissions]; S7[Reports]; S8[Assistant]; S9[Ops: backup/logs/release-notes]
    end
    KIT[Generic kits: Search · Import · FilterSchema]
    ABS[Abstractions / ports]
  end
  subgraph Dom[Outlay.Domain]
    ENT[Entities · Value Objects · Domain events · Rules]
  end
  subgraph Inf[Outlay.Infrastructure]
    P1[Persistence: InMemory · EfCore · Dapper · Mongo]
    C1[Caching: HybridCache · Redis · Memcached]
    M1[Messaging: SMS · Email · Outbox]
    L1[Logging: Channel · FileWriter · Redactor · Advisor · Query interceptors]
    B1[Backup providers]
    I1[Import workers]
  end
  MW --> EP --> BEH --> Slices --> ABS
  Slices --> KIT
  Slices --> ENT
  ABS -. implemented by .-> Inf
  OID --> EP
  L1 -. fed by .-> MW & BEH & P1
```

## Level 3 — Frontend components
```mermaid
flowchart LR
  Pages[app/[locale] pages] --> Features[features/*]
  Features --> UI[components/ui kit: DataTable · AdvancedFilterBar · BulkImportDialog · …]
  Features --> Hooks[TanStack Query hooks]
  Hooks --> Http[lib/http: auth · correlation · idempotency · locale]
  Hooks --> Off[lib/offline: Dexie · outbox · sync]
  Http --> BFF[app/api/session BFF] --> API[(Outlay API)]
  SW[service-worker.js] --- Pages
  Off --- IDB[(IndexedDB)]
  I18n[next-intl en/bn] --- Pages
```

## Dynamic — Log pipeline
```mermaid
sequenceDiagram
  participant H as Handler / Middleware / DB interceptor
  participant L as IOutlayLogger
  participant R as Redactor
  participant Q as Channel
  participant W as FileWriter
  H->>L: Info/Error/Exception/Query/Audit (+caller class,file,method,line)
  L->>R: mask sensitive keys & patterns
  R->>Q: LogEntry
  Q->>W: dequeue (single reader)
  W->>W: logs/<category>/dd-MM-yyyy.txt (if category enabled)
  W-->>Console: red if slow query
```

## Deployment view
```mermaid
flowchart TB
  Internet --> NG[Nginx LB :443]
  NG --> W1[web x1..n]
  NG --> A1[api #1] & A2[api #2]
  A1 & A2 --> PG[(PostgreSQL / chosen provider)]
  A1 & A2 --> RD[(Redis)]
  A1 & A2 --> VOL[(logs/ & backups volume)]
  subgraph Dev profile
    IM[Single API + InMemory DB]
  end
```
Note: InMemory profile = **one** API container (A-07). Compose profiles add `mssql`, `mysql`, `oracle`, `mongo`, `mail`.

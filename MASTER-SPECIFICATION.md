# Outlay — Master Specification

| Field | Value |
|---|---|
| Product | Outlay: multi-tenant Expense Tracking & Budget Control SaaS |
| Version | 0.0.0 (specification baseline) · 2026-10-28 |
| Status | Draft for owner review. Open questions in §6. |
| Owner | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

This is the **single entry point** to the Outlay specification. It shows where every requirement lives, how conflicts between documents are resolved, and what is still undecided. Read this first, then the document you need.

---

## 1. Document Map

| # | Document | Answers | Audience |
|---|---|---|---|
| 1 | [docs/PRD.md](docs/PRD.md) | **What** we build and why: roles, plans, features (`FR-*`), business rules (`BR-*`), NFRs, user stories | Everyone |
| 2 | [docs/AppFlow.md](docs/AppFlow.md) | **How users move**: routes, step-by-step flows (`F-*`), failure branches | Product, frontend, QA |
| 3 | [docs/UiSpec.md](docs/UiSpec.md) | **How it looks**: "Kantha Ledger" design system, tokens, components, every screen (`S-*`), i18n and a11y rules | Frontend, design |
| 4 | [docs/TRD.md](docs/TRD.md) | **How it works**: architecture, C4, stack, security, logging, caching, search/import/assistant engines, API catalogue, testing, DevOps | Engineering |
| 5 | [docs/Schema.md](docs/Schema.md) | **What is stored**: ER diagram, tables, indexes, provider type mapping, seeds | Backend, DBA |
| 6 | [docs/Plan.md](docs/Plan.md) | **In what order**: milestones M0–M13, tasks, Definition of Done | Engineering, AI agent |
| 7 | [docs/adr/](docs/adr/) | **Why** each major decision was made (ADR-0001…0018) | Engineering |
| 7a | [docs/diagrams/C4-DIAGRAMS.md](docs/diagrams/C4-DIAGRAMS.md) | **Pictures** of the architecture: C4 context, containers, components (API + web), dynamic, deployment, code | Everyone |
| 7b | [docs/guides/BACKEND-DEVELOPER-GUIDE.md](docs/guides/BACKEND-DEVELOPER-GUIDE.md) | **How to work on the backend**: setup, architecture tour, build-a-CRUD-slice walkthrough, how-tos, testing, debugging, upgrades | Backend developers, AI agent |
| 7c | [docs/guides/FRONTEND-DEVELOPER-GUIDE.md](docs/guides/FRONTEND-DEVELOPER-GUIDE.md) | **How to work on the frontend**: setup, tokens, data layer, add-a-CRUD-screen walkthrough, offline, i18n, PWA, testing | Frontend developers, AI agent |
| 8 | [CLAUDE.md](CLAUDE.md) | **Rules for the AI pair programmer** (the "AI brain") | AI agent, developers |
| 9 | [MASTER-PROMPT.md](MASTER-PROMPT.md) | Prompts to drive an AI coding agent through the plan | Developer |
| 10 | [ROADMAP.md](ROADMAP.md) | Releases v0.5 → v2.0 | Stakeholders |
| 11 | [RELEASENOTES.md](RELEASENOTES.md) | What shipped and what SQA must test (mirrored by the Release Notes API) | SQA, stakeholders |
| 12 | [AI-HANDOVER.md](AI-HANDOVER.md) | Current state and next step, for whoever (human or AI) continues | Next session |

### 1.1 Precedence when documents disagree

1. **Owner's explicit decision** (recorded as a resolved question in §6 or an ADR)
2. **PRD** for behaviour and business rules
3. **ADRs** (newest first) for technical decisions
4. **TRD / Schema / UiSpec / AppFlow** for implementation detail
5. **Plan** for sequencing only

Whoever finds a conflict fixes the lower-precedence document in the same PR and notes it in `AI-HANDOVER.md`.

---

## 2. Product in One Page

- **Who:** organisations (tenants) that sign up, choose a plan (Starter / Pro / Enterprise), invite their team, set **yearly budgets per category** (each one a monthly allowance, ADR-0014), and track expenses. Four roles: **Super Admin** (platform operator, never sees expense data), **Admin**, **Manager**, **Member/User**.
- **Core loop:** a member adds an expense → the system checks the budget → it's saved (Soft limit, flagged if over) or **held for approval** (Hard limit) → Managers/Admins approve or reject → the dashboard, monthly summary and alerts (80% once per month, over budget) stay current.
- **Extended engineering scope (owner brief):** OAuth 2.0 / OIDC with OpenIddict, BCrypt, MFA (dev OTP `111111`), lockout after 3 failures, delegated permissions with a dedicated UI, advanced Jira-style search and all-or-nothing CSV/Excel import on every CRUD, offline-first PWA with IndexedDB sync, EN/বাংলা localisation, a deterministic voice/text assistant with tenant guardrails, categorised logs with slow-query detection, 7 switchable DB providers, caching with Redis, idempotency, correlation IDs, resilience, rate limiting, load balancing, Docker, CI/CD, backup/log/release-notes APIs, Scalar + Postman, and 5 kinds of automated tests.

---

## 3. Architecture in One Page

```
React 19 PWA (Tailwind v4, TanStack Query, Dexie outbox, i18next, service-worker.js)
        │  HTTPS / WSS  (Bearer access token in memory, refresh token in HttpOnly cookie)
        ▼
nginx load balancer (TLS, least_conn, edge rate limit)
        ▼
Outlay.Api × N  (.NET 10, Minimal APIs per slice)
   Diagnostics middleware → RateLimiter → OpenIddict validation → Endpoint (+Idempotency)
   → MediatR pipeline: Logging → Validation → Authorization → Tenant/Plan → Caching → Transaction
   → Handler (Vertical slice) → Domain (rich entities, rules) → IAppDbContext / read stores
        │                    │                         │
   EF Core 10 / Dapper /   FusionCache L1 + Redis L2   Background workers (Channels):
   Mongo driver            + backplane; SignalR        imports, outbox, aggregates, backups
        ▼                  backplane
InMemory | SQL Server | PostgreSQL | MySQL | Oracle | MS Access | MongoDB   (Database:Provider)
Logs: logs/{application|application-error|application-exceptions|application-query|application-audit}-logs/dd-MM-yyyy.txt
```

**Key decisions:** Clean Architecture + Vertical Slices (ADR-0002) · CQRS/MediatR (ADR-0003) · OpenIddict + Identity (ADR-0005) · BCrypt (ADR-0006) · delegated subset-only permissions (ADR-0007) · shared DB with TenantId filters (ADR-0008) · tiered multi-provider persistence (ADR-0009) · FusionCache (ADR-0010) · categorised Serilog (ADR-0011) · IndexedDB outbox (ADR-0012) · deterministic assistant (ADR-0013).

---

## 4. Requirements Traceability Matrix

Every item in the owner's brief, mapped to where it is specified and when it is built.

| ID | Owner requirement | Specified in | Built in |
|---|---|---|---|
| R-01 | Six spec docs following the "spec before code" pattern: PRD, AppFlow, UiSpec, TRD, Schema, Plan | docs/*.md | — |
| R-02 | MASTER-SPECIFICATION, MASTER-PROMPT, ROADMAP, RELEASENOTES, AI-HANDOVER, CLAUDE.md | root *.md | — |
| R-03 | Requirement PDF basics: multi-tenant, 4 roles, 3 plans, 10 core features, 8 screens, flows, business rules, example data, US-20/21/22 | PRD §2–§10, AppFlow F-01…F-08, UiSpec S-01…S-17 | M2–M6 |
| R-04 | Clean Architecture, Vertical Slice, CQRS with MediatR | TRD §1, §4, §5; ADR-0002/3/4 | M0 |
| R-05 | OAuth 2.0, Microsoft Identity, OpenIddict, RBAC | TRD §7.1, §7.3; ADR-0005/7 | M1, M5 |
| R-06 | Logs configurable, on/off via appsettings.json or .env | TRD §8.2 | M0-07 |
| R-07 | 5 log types at `logs/<type>/dd-MM-yyyy.txt` | TRD §8.1 (see Q-07) | M0-07 |
| R-08 | Custom middleware; logs show class, file, location, method, line; exact reason and possible fixes | TRD §8.4 (ExceptionAdvisor) | M0-08 |
| R-09 | Query log: threshold, red colour, class/file/method/line, generated query, start, end, execution time | TRD §8.5 | M0-11 |
| R-10 | DB abstraction switchable in appsettings/.env: InMemory, MS SQL, MySQL, Oracle, Postgres, MS Access, MongoDB; EF Core (+Dapper optional) | TRD §6; Schema §5; ADR-0009 | M0-10, M12 |
| R-11 | React latest, upgradable; .NET 10 → 11+ hassle-free | TRD §3, §18 | M0-02, CI next job |
| R-12 | Advanced multi-field search on every CRUD (equals, starts with, ends with, contains, not contains, …) | PRD FR-SRCH; TRD §13.1; UiSpec §6.3 | M7 |
| R-13 | Bulk insert from CSV and Excel, all row/column errors and duplicates pointed out, full rollback, easy re-upload | PRD FR-IMP; TRD §13.2; AppFlow F-10; ADR-0015 | M7 |
| R-14 | Work in different threads for speed | TRD §11.1 | M7, M8 |
| R-15 | Redis backplane; serve from cache before DB; fall back to memory/Memcached when Redis is off | TRD §10; ADR-0010 | M8-01/02 |
| R-16 | Frontend keeps data and CRUDs in IndexedDB when the backend is down; syncs when online | PRD FR-OFF; TRD §12.3; AppFlow F-11; ADR-0012 | M9 |
| R-17 | DB pooling | TRD §6.3 | M0-10 |
| R-18 | Idempotency keys against duplicate transactions | TRD §11.3; Schema `idempotency_records` | M8-03 |
| R-19 | Correlation IDs | TRD §8.4 | M0-08 |
| R-20 | Resilience retry policy configurable in appsettings/.env | TRD §11.4 | M8-04 |
| R-21 | Token rotation and revocation; JWT and tokens kept secure | TRD §7.1 | M1-02/07 |
| R-22 | Passwords hashed with BCrypt ("and encrypted", see note) | TRD §7.2; ADR-0006 | M1-01 |
| R-23 | No sensitive data in logs (passwords, money amounts, …) | TRD §8.3 masking; BR-SEC-01 | M0-07 |
| R-24 | CI/CD pipelines | TRD §17.3 | M0-21, M13-09 |
| R-25 | Dockerised properly | TRD §17.1 | M0-20 |
| R-26 | Load balancing | TRD §11.6 | M8-06 |
| R-27 | Rate limiting | TRD §11.5 | M1-10, M8-05 |
| R-28 | Audit log with the user's IP address | TRD §8.6; Schema `audit_entries` | M2-09 |
| R-29 | MFA/2FA: dev OTP `111111`; prod real OTP via SMS **and** email; countdown; valid 5 minutes | PRD FR-AUTH-05/06; TRD §7.6; UiSpec S-02 | M1-04/05 |
| R-30 | Free SMS gateway wired; Twilio configurable via env/appsettings | TRD §7.7 (see Q-05) | M1-06 |
| R-31 | Lock user after 3 wrong passwords | PRD FR-AUTH-03; AppFlow F-02 | M1-03 |
| R-32 | Password reset and change password | AppFlow F-02b | M1-09 |
| R-33 | Every endpoint documented with a proper title | TRD §14.3 (build-breaking test) | M0-13 |
| R-34 | Postman collection with pre- and post-scripts, role-based (Super Admin, Admin, Manager, User) | TRD §14.4 | M1-13, M5-06, M13-03 |
| R-35 | Scalar OpenAPI docs with a colourful, charming theme | TRD §14.3; UiSpec §11 | M0-13 |
| R-36 | API to download a DB backup for whichever provider is active | TRD §13.5 | M11-03 |
| R-37 | API to download logs by category and date `dd-MM-yyyy` | TRD §13.6 | M11-02 |
| R-38 | Release notes API for SQA (what's new, what to check) | TRD §13.7; RELEASENOTES.md | M0-22, M11-04 |
| R-39 | Optional session-time field on the login API (hours/days), absent in the UI | PRD FR-AUTH-09; TRD §7.1 | M1-08 |
| R-40 | Dedicated permissions API; Super Admin → Admin → Manager → User delegation, subset only | PRD §3.3; TRD §7.3; ADR-0007 | M5 |
| R-41 | Dedicated permissions UI page | UiSpec S-16 | M5-05 |
| R-42 | Tailwind CSS; charming, unique, professional design | UiSpec §1–§4 | M0-15/16 |
| R-43 | Offline-first, responsive, PWA with `manifest.json` + `service-worker.js` and install prompt | UiSpec §3, §9; TRD §12.4 | M0-18, M9 |
| R-44 | Localised EN/BN in backend and frontend: whole screen, titles, numbers, pagination | PRD FR-I18N; UiSpec §8; TRD §15 | M0-14/17 |
| R-45 | Unit, integration, performance, load and stress tests as separate projects | TRD §16; ADR-0017 (xUnit + NSubstitute + AwesomeAssertions, Testcontainers, NetArchTest, BenchmarkDotNet, k6 with failure injection) | M0-03, M8-08 |
| R-46 | README, user guide, backend and frontend developer guides (how to create a CRUD) | docs/guides/BACKEND-DEVELOPER-GUIDE.md, FRONTEND-DEVELOPER-GUIDE.md (first versions done); README + USER-GUIDE in Plan M13-04/05 | Guides drafted; finalised M13 |
| R-47 | ADRs, C4 diagrams, ER diagrams, schema designs, architecture docs | docs/adr; docs/diagrams/C4-DIAGRAMS.md; Schema §2 | Done (baseline), SVG export M13-08 |
| R-48 | Simple, maintainable code; OOP, OOAD, SOLID, KISS, YAGNI, DRY | TRD §1; CLAUDE.md | Every task (DoD) |
| R-49 | CLAUDE.md "AI brain" | CLAUDE.md | Done (baseline) |
| R-50 | Voice and text assistant; deterministic; multi-tenant guardrails (Admin can ask about their Managers/Users, never other companies or Super Admin) | PRD FR-AST; TRD §13.3; AppFlow F-13; ADR-0013 | M10 |
| R-51 | When Redis/DB is down: graceful backend handling and a friendly message with a configurable support name, phone and email | TRD §9; AppFlow F-15 | M8-07 |
| R-52 | Deliver as a zip | — | Done |

---

## 5. Glossary

| Term | Meaning |
|---|---|
| Tenant / Organisation / Workspace | One customer company. Data boundary. |
| Platform tenant | Pseudo-tenant that holds Super Admin memberships |
| Membership | A user's role in one tenant. A user can have several. |
| Role template | Default permission set: SuperAdmin, TenantAdmin ("Admin" in UI), Manager, Member ("User" in the owner's brief) |
| Permission | Atomic capability, e.g. `expenses.approve` |
| Data scope | `own` / `team` (my subtree) / `all` (tenant) |
| Soft limit / Hard limit | Over budget → saved and flagged / held for approval |
| Spent | Sum of Saved + Approved expenses ± adjustments in the period |
| Adjustment | Separate correction record; the original expense is never modified |
| OQL | Outlay Query Language: the text form of advanced search |
| Outbox | (Server) table of messages to deliver; (client) IndexedDB queue of offline mutations |
| Step-up | A fresh OTP required for sensitive actions |
| Tier (DB) | Level of support and testing for a database provider |

---

## 6. Open Questions & Assumptions (owner decisions needed)

| ID | Question | Current assumption (used in the specs) | Impact if changed |
|---|---|---|---|
| ~~Q-01~~ | ~~Yearly or monthly budget?~~ **Resolved 2026-10-27:** the owner had no preference, so the recommended option stands. The yearly record holds a **monthly allowance** applied to every month of that year, and alerts run monthly (ADR-0014, Accepted). | — | — |
| ~~Q-02~~ | ~~When does editing stop?~~ **Resolved 2026-10-28:** the owner kept the working assumption. Own **Saved** or **Pending** expenses can be edited or deleted while their month is open (until the month-close snapshot on the 1st + `Reports:MonthCloseGraceDays`, default 3). **Every edit re-runs validation, the duplicate check (excluding itself) and the budget check (excluding its old amount)**, so an edit can move Saved→Pending or Pending→Saved. Approved/Rejected entries, closed months and previous years are locked, and corrections go through Adjustments (BR-EXP-05, BR-EXP-08…13). | — | — |
| ~~Q-03~~ | ~~Negative remaining amount?~~ **Resolved 2026-10-28:** shown as **"Over by ৳2,100"** in red with an "Over budget" label, never with a minus sign. The Total row keeps the arithmetic total (৳1,50,000 − ৳86,400 = ৳63,600). | — | — |
| ~~Q-04~~ | ~~Manager scope?~~ **Resolved 2026-10-28:** Managers default to the **whole organisation** (`expenses.read.all`), matching the brief's screens. An Admin can narrow a Manager to their **team** (hierarchy subtree) on the Permissions page. The assistant follows the same scope. **Approvals follow it too:** approvers are all Admins plus Managers with `expenses.approve` whose scope covers the member. A team-only Manager doesn't receive or see requests from outside the team, and gets 404 if they try to approve one (BR-NOT-04/05). | — | — |
| **Q-05** | "Free SMS gateway": there's no reliable free production gateway. Which provider for Bangladesh production? | Console (dev), TextBelt free key (smoke tests, ~1/day), Twilio (prod). Email OTP is always sent too. | Cost, deliverability |
| ~~Q-06~~ | ~~MediatR licence?~~ **Resolved 2026-10-28:** the owner's company qualifies, so Outlay uses **MediatR 14.x with the free Community licence key**, stored in secrets as `MediatR:LicenseKey`. It's the current, fully patched release, not the old 12.5.0. Re-check eligibility if the company passes US$5M revenue or US$10M outside capital (ADR-0003). | — | — |
| **Q-07** | The error-log filename in the brief is `dd-mmyyyy.txt` (no dash). Intentional? | `dd-MM-yyyy.txt` for all categories | File naming |
| **Q-08** | Should the DB audit table store monetary old/new values (financial audit) even though log **files** must not? | DB audit masks amounts by default. `Audit:StoreSensitiveValues=true` enables it. | Compliance |
| **Q-09** | DB backup contains every tenant's expense data, but Super Admin "cannot see expense data". Allow backup download? | Allowed as an operations exception: Super Admin only, step-up OTP, audited, `system.backup.download` permission | Policy |
| **Q-10** | Rejected Hard-limit entries are "permanently discarded". Keep a hidden tombstone for audit? | Yes: status `Rejected`, excluded from everything, retained for audit | Data retention |
| **Q-11** | Billing/payment integration for Pro ($49) | Out of scope for v1. Plan changes go through Super Admin or a stubbed upgrade request. | Scope |
| **Q-12** | Are Tier-3 providers (MS Access: Windows-only; MongoDB: limited EF support) acceptable as "experimental"? | Yes, documented as limited | Effort |
| ~~Q-13~~ | ~~Does the existing repo contain code to keep?~~ **Resolved 2026-10-27:** the repo (commit `b5fc8f3`) is a fresh scaffold: root `Outlay.slnx`, `src/Backend/Outlay.{Domain,Application,Infrastructure,Api}` (.NET 10, correct references, WeatherForecast template) and `src/Frontend/outlay-web` (Vite 8, React 19.3, TS 6). Specs now use this layout (TRD §4). | — | — |
| **Q-14** | The YouTube short's exact template couldn't be viewed (page content not retrievable). Do the 6 core docs need a specific section layout from the video? | Standard spec-driven layout: PRD → AppFlow → UiSpec → TRD → Schema → Plan | Doc format |

---

## 7. Limitations & Deviations from the Brief

Everything in the owner's brief is specified. These are the places where a requirement **can't be met exactly** because of a technical limit, or where the spec **deliberately deviates** from the wording. Each links to the section that handles it.

### 7.1 Technical limits

| # | Brief item | Limitation | How Outlay handles it | Where |
|---|---|---|---|---|
| L-01 | Free SMS gateway | No free gateway is reliable for production. The TextBelt free key sends about 1 SMS/day. | Console (dev), TextBelt (smoke tests), Twilio (production). OTP is **always also sent by email**. | TRD §7.7, Q-05 |
| L-02 | MS Access provider | Needs the Windows-only ACE OLE DB driver, so it can't run on Linux or Docker | Tier 3 (experimental), excluded from Linux builds by an MSBuild switch | TRD §6.2 |
| L-03 | MongoDB provider | The EF Core provider has no `Select` projections, migrations or foreign keys | `ProjectSafe`, a MongoDB-driver read store for reports, Tier 3 | TRD §6.2, §6.4 |
| L-04 | Dapper | Works with SQL databases only | Used for relational report reads only. MongoDB and InMemory use EF/driver paths. | TRD §6.4 |
| L-05 | MySQL with EF Core 10 | Pomelo has no EF Core 10 build | Oracle's `MySql.EntityFrameworkCore` (fork as fallback) | ADR-0009 |
| L-06 | Red slow-query logs | A `.txt` file can't hold colour | ANSI red on the console, `[SLOW QUERY ⚠]` prefix in the file, red rows in the S-22 log viewer | TRD §8.5 |
| L-07 | Voice assistant | Firefox has no speech recognition. Bengali recognition quality depends on the browser's cloud service. Deterministic means fixed phrasing coverage. | Feature detection with a text fallback, privacy note, suggestions for unmatched questions | TRD §12.6, §13.3 |
| L-08 | PWA install notification | iOS Safari has no install-prompt API. Background Sync is Chromium-only. | iOS "Add to Home Screen" hint. Sync on `online`/focus/interval as fallback. | UiSpec §9, TRD §12.4 |
| L-09 | DB backup for every provider | Native tools need client binaries or a shared volume. Oracle Data Pump needs server directories. InMemory has no native backup. | Native where possible, otherwise a portable logical JSON export | TRD §13.5 |
| L-10 | Logs download API | With several API replicas, each writes its own files | Instance subfolders or a shared volume. Central log sink recommended. | TRD §11.6, §13.6 |
| L-11 | Token revocation | JWT access tokens can't be recalled from clients | Security-stamp and permission-version check, so revocation takes effect within ≤ 60 s | TRD §7.1 |
| L-12 | Hassle-free .NET 11+ upgrade | Depends on third-party packages (EF providers) shipping matching versions | Central package versions, one TFM property, per-provider MSBuild switches, CI job on the next .NET | TRD §18 |
| L-13 | InMemory database + load balancing | InMemory data lives in one process | InMemory requires exactly one API instance (startup guard, single-container compose profile) | TRD §6.2, §11.6, C4 Deployment |

### 7.2 Deliberate deviations from the wording

| # | Brief wording | Spec decision | Why | Where |
|---|---|---|---|---|
| D-01 | Passwords "hashed and encrypted by bcrypt" | BCrypt **hash** only | Encryption is reversible and would weaken password storage | ADR-0006 |
| D-02 | Error log file `dd-mmyyyy.txt` | `dd-MM-yyyy.txt` like the other four logs | Consistency (Q-07) | TRD §8.1 |
| D-03 | "No money amounts in logs" | Log files always mask amounts. DB audit masks by default, with a switch for financial audit. | Financial audits may need values. Files never contain them. (Q-08) | TRD §8.6 |
| D-04 | "Don't hit the DB if the data is in Redis" | Applies to reads. Writes always go to the DB. | The cache must never be the system of record | TRD §10 |
| D-05 | "All work in different threads" | Async I/O end to end, background workers for long jobs, parallel CPU work for imports | That's what actually makes .NET fast. A thread per operation would be slower. | TRD §11.1 |
| D-06 | Optional session time on the login API | Implemented on OAuth's password grant for first-party clients only | OAuth 2.1 discourages the password grant for third-party apps | ADR-0005 |
| D-07 | Super Admin can't see expense data, but can download DB backups | Backups allowed with step-up OTP + audit | Operations need backups (Q-09) | TRD §13.5 |
| D-08 | "Negative amounts never appear" vs. the mock's "−৳2,100" | Shown as "Over by ৳2,100" | Follows the written rule (Q-03) | UiSpec S-14 |

### 7.3 Licences to be aware of

| Package | Status | Decision |
|---|---|---|
| MediatR 13+ | Dual RPL-1.5/commercial. Free Community licence under US$5M revenue. | Used: MediatR 14.x with the Community licence (owner's company qualifies, Q-06 resolved). |
| AutoMapper 15+ | Commercial. 14.0.0 has unpatched CVE-2026-32933. | Not used. Manual mapping (ADR-0016). |
| FluentAssertions 8+ | Commercial | Not used. AwesomeAssertions (ADR-0017). |
| NBomber | Paid for organisational use | Not used. k6 (ADR-0017). |

---

## 8. Constraints & Non-Negotiables

1. **Tenant isolation is absolute.** Any code path that could cross tenants needs an isolation test.
2. **No sensitive values in log files:** passwords, OTPs, tokens, secrets, card numbers, money amounts.
3. **Dev OTP `111111` exists only in Development**, and the app refuses to start otherwise.
4. **Every endpoint is documented** (summary, description, tags, examples) and **covered** by integration and Postman tests.
5. **Every UI string exists in English and Bengali.**
6. **Keep it simple:** no microservices, no event sourcing, no generic repository over EF Core, no abstraction without a second implementation or a hard requirement.

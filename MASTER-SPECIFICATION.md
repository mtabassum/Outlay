# Outlay — Master Specification

> **Single source of truth.** If any other document disagrees with this file, this file wins and the other document is fixed.
> Product: **Outlay** — Multi-tenant Expense Tracking & Budget Control SaaS.
> Doc version: 1.0 · Status: Approved for build · Author: MD IKRAMUL ISLAM SIDDIQUE POROSH

---

## 1. How the documents fit together

| Document | Answers | Audience |
|---|---|---|
| `MASTER-SPECIFICATION.md` (this) | *What exactly must exist* — requirement IDs, rules, decisions | Everyone |
| `PRD.md` | *Why* and *for whom* | Product, QA, stakeholders |
| `AppFlow.md` | *How a user/system moves* through the product | Product, FE, BE, QA |
| `UiSpec.md` | *What each screen looks like and does* | FE, design, QA |
| `TRD.md` | *How it is built technically* | BE, FE, DevOps |
| `Schema.md` | *Data model, ER diagram, DB-provider mapping* | BE, DBA |
| `Plan.md` | *Order of work, milestones, definition of done* | Developers, AI agents |
| `MASTER-PROMPT.md` | *Copy-paste prompt that builds the system* | AI agents |
| `ROADMAP.md` | *Releases over time* | Everyone |
| `RELEASENOTES.md` | *What changed + what QA must verify* (also served by API) | QA, stakeholders |
| `AI-HANDOVER.md` | *Where we stopped and the exact next command* | Next AI/developer |
| `CLAUDE.md` | *AI "brain": rules the AI must always follow* | AI agents |
| `docs/*` | Developer guides, ADRs, C4 diagrams | Developers |

## 2. Glossary

| Term | Meaning |
|---|---|
| Tenant / Organisation | A company workspace. All its data is private. |
| Membership | A user's role + permissions inside one tenant. One user may have many. |
| Super Admin | Platform operator. Not part of any tenant. **Cannot read tenant expense data.** |
| Admin | Tenant Admin (owner / IT lead). |
| Manager | Team lead / finance officer. |
| Member | Regular employee ("user"). |
| Soft Limit / Hard Limit | Budget behaviour at 100% (§5 BR-BUD). |
| Adjustment | Append-only correction of an expense. Originals are never edited. |
| Permission | A `resource:action` string, e.g. `expenses:approve`. |
| Permission ceiling | A user can only grant permissions they themselves hold. |

## 3. Roles & hierarchy

```
Super Admin ──creates──► Admin ──creates──► Manager
                           └────creates──► Member
```
* Only **Super Admin** creates Admins (tenant provisioning). Only **Admin** creates/removes Managers and Members (spec §8: "Only the Tenant Admin can invite users, change roles, or remove members").
* Every grant is capped by the grantor's own permissions (**permission ceiling**, ADR-0010). Revoking a permission from a grantor cascades to everything they delegated.
* Super Admin has **platform** permissions only; tenant-scoped endpoints reject a token that has no tenant context.

## 4. Requirement catalogue

Release column: **R1** 0.1 Foundation MVP · **R2** 0.2 Governance · **R3** 0.3 Scale & Portability · **R4** 0.4 Offline & Intelligence · **R5** 1.0 Production hardening. See `ROADMAP.md`.

### 4.1 Organisation & tenancy
| ID | Requirement | Rel |
|---|---|---|
| FR-ORG-01 | Org sign-up (name, work email, password) creates a private workspace; signer becomes Admin. | R1 |
| FR-ORG-02 | Onboarding wizard: 1 Create account → 2 Set up org (name, industry, team size, currency) → 3 Choose plan → 4 Invite team (also guides categories → budgets). | R1 |
| FR-ORG-03 | Strict tenant isolation on DB reads/writes, cache keys, search, bulk import, logs, assistant. | R1 |
| FR-ORG-04 | One person, many orgs, one login; workspace selector after login. | R1 |
| FR-ORG-05 | Org currency (default BDT ৳); decimals supported. | R1 |

### 4.2 Authentication, security, permissions
| ID | Requirement | Rel |
|---|---|---|
| FR-AUTH-01 | Login with email + password + workspace context. | R1 |
| FR-AUTH-02 | OAuth 2.0/OIDC with **OpenIddict**; JWT access tokens; **refresh-token rotation with reuse detection**; revocation endpoint; logout revokes. | R1 |
| FR-AUTH-03 | Passwords hashed with **BCrypt** (work factor ≥ 12, configurable). Other secrets at rest (TOTP seeds, phone numbers if configured) encrypted with AES-GCM via Data Protection. | R1 |
| FR-AUTH-04 | **3 consecutive wrong passwords lock the account.** Unlock by password reset, Admin/Super-Admin unlock, or optional auto-unlock (`Security:Lockout:AutoUnlockMinutes`, default 0 = manual only). | R2 |
| FR-AUTH-05 | Forgot password (OTP-verified reset) and change password (needs current password; revokes other sessions). | R2 |
| FR-AUTH-06 | MFA/2FA by OTP: **dev/test fixed `111111`**; other environments random 6-digit, valid **5 min**, UI countdown, resend cooldown 60 s, max 5 attempts, stored hashed. Delivery by SMS and email. | R2 |
| FR-AUTH-07 | SMS: a **free gateway adapter** and **Twilio** both wired; selected by config; mock provider for dev. | R2 |
| FR-AUTH-08 | Login API accepts **optional `sessionMinutes`** (bounded by `Security:Session:MaxMinutes`). The UI never shows this field. | R2 |
| FR-AUTH-09 | Dedicated **Permissions API + UI page** implementing delegation with ceiling & cascade (§3). | R2 |
| FR-AUTH-10 | RBAC policies `[HasPermission("x:y")]` on every endpoint; default deny. | R1 |
| FR-AUTH-11 | Rate limiting (global, per-user, strict on login/OTP); IP captured via trusted forwarded headers. | R3 |

### 4.3 Subscription plans
| ID | Requirement | Rel |
|---|---|---|
| FR-PLAN-01 | Plans: **Starter** (free; 3 users; 10 categories; 6-month history visible; basic monthly report; in-app notifications), **Pro** ($49/mo; 15 users; unlimited categories; full history; advanced report + member filter; email & SMS alerts; manager approval workflow), **Enterprise** (custom; unlimited; archive/data export; custom reports; SSO; priority support). | R1 (Starter/Pro limits) |
| FR-PLAN-02 | Enforce: 4th user on Starter blocked with upgrade prompt; 11th category blocked; history older than 6 months **hidden not deleted**; downgrade refused if usage exceeds target limits. | R1 |
| FR-PLAN-03 | Limits are data (table `SubscriptionPlans`), not hard-coded. Billing is a stub in v1 (plan selection only; no payment gateway). | R1 |

### 4.4 Categories, budgets, expenses
| ID | Requirement | Rel |
|---|---|---|
| FR-CAT-01 | Admin creates/edits/deactivates/reactivates categories. Unique per tenant case-insensitively; non-blank; ≤ 50 chars. | R1 |
| FR-CAT-02 | Deactivated categories hidden from entry screen, still shown in reports/history. | R1 |
| FR-BUD-01 | One budget per category per year per tenant; decimals allowed; only current year editable; past years locked. | R1 |
| FR-BUD-02 | Limit type Soft/Hard per budget (Soft only in R1). No budget ⇒ Unlimited, no alerts. | R1 / Hard R2 |
| FR-EXP-01 | Add expense: date, category, amount, description. Amount > 0; date not in future; category mandatory. | R1 |
| FR-EXP-02 | Duplicate warning (same date+amount+category+description for same tenant/user); warns, never blocks. | R1 |
| FR-EXP-03 | Members see/edit/delete only **own** expenses of the **current year**; Managers/Admins see all team expenses. | R1 |
| FR-EXP-04 | Adjustment entries for corrections/refunds; original never modified; reports use net effect; **negative amounts never displayed** (adjustments show as "Adjustment −" with positive magnitude + direction label). | R2 |
| FR-EXP-05 | Statuses: `Saved`, `OverBudget`, `PendingApproval`, `Approved` (hard-limit approved), plus adjustment rows. Rejected = discarded from ledger; an immutable approval-request record keeps who/when/why. | R2 |

### 4.5 Budget monitoring, approvals, notifications, reports
| ID | Requirement | Rel |
|---|---|---|
| FR-MON-01 | Every new expense is checked against category budget. 80% alert **once per category per month**. 100%: Soft ⇒ saved + flagged "Over Budget" + alert to Admins & Managers; Hard ⇒ `PendingApproval`. | R1 (80% alert), R2 (100%) |
| FR-APR-01 | Managers/Admins approve (optional note) or reject (reason mandatory). Approved ⇒ saved with approver + time; member notified. Rejected ⇒ discarded; member notified with reason. | R2 |
| FR-NOT-01 | In-app notifications on by default: thresholds, over-budget, approval requests (sent immediately to all Managers/Admins), new member joined, seat usage. De-duplicated per event per period. Mark-all-read. | R1 |
| FR-NOT-02 | Email & SMS alerts on Pro/Enterprise. | R2 |
| FR-RPT-01 | Monthly Summary (Manager/Admin): total spent, total budget, remaining, over-budget count, per-category table with status chips, member filter, month filter, member-wise view, export. | R1 (basic) R2 (member filter/export) |
| FR-RPT-02 | Dashboard (Admin): spent, total budget, seats used, pending approvals, budget-health bars, recent expenses. | R1 |
| FR-SA-01 | Super Admin platform dashboard: org list (plan, users x/limit, status, joined), plan management. No access to expense data. | R1 |

### 4.6 Cross-cutting CRUD capabilities (apply to **every** CRUD)
| ID | Requirement | Rel |
|---|---|---|
| FR-SRCH-01 | **Advanced search** (Jira/GitLab-style): multi-field filters, operators `eq, neq, contains, notContains, startsWith, endsWith, gt, gte, lt, lte, between, in, notIn, isNull, isNotNull`, AND/OR groups, sort, paging, saved filters. Fields allow-listed per entity. | R2 |
| FR-BULK-01 | **Bulk insert** from CSV and Excel (.xlsx). Template download. Validate every row; all-or-nothing transaction; on failure return **every** problem (row, column, code, localized message, offending value redacted if sensitive) plus downloadable error report; duplicates inside the file and against DB are both reported. | R2 |
| FR-BULK-02 | Imports run on a background worker with job status polling (non-blocking). | R2 |

### 4.7 Platform capabilities
| ID | Requirement | Rel |
|---|---|---|
| FR-DB-01 | DB abstraction; default **InMemory**; switch via `appsettings.json` / `.env` between InMemory, MS SQL, MySQL, Oracle, PostgreSQL, MS Access, MongoDB. ORM switch: **EF Core** or **Dapper** (Mongo: official driver). | R1 InMemory, R3 rest |
| FR-CACHE-01 | Read-through cache; Redis (with backplane/pub-sub invalidation) → fallback to in-process → (optional Memcached). Never hit DB if value is cached. Graceful when Redis is off. | R3 |
| FR-RES-01 | Configurable retry/circuit-breaker/timeouts (Polly v8 / `Microsoft.Extensions.Resilience`). | R3 |
| FR-IDEM-01 | `Idempotency-Key` header on creating POSTs; duplicate key returns the original result. | R1 |
| FR-CORR-01 | `X-Correlation-Id` accepted/generated, echoed, in every log line and error body. | R1 |
| FR-DEG-01 | If Redis/DB/SMS down: backend degrades gracefully; API returns RFC 7807 problem with support contact (name, phone, email from config); frontend shows localized "Something went wrong — please contact Outlay support" message. | R1 |
| FR-OPS-01 | **Backup API** (Super Admin): downloads backup of the *active* provider. | R3 |
| FR-OPS-02 | **Log download API** (Super Admin): by category + date `dd-MM-yyyy`. | R1 |
| FR-OPS-03 | **Release-notes API**: current & previous releases with "what to verify" checklist for QA. | R1 |
| FR-OPS-04 | Health endpoints: `/health/live`, `/health/ready` (checks DB, Redis, SMS/SMTP). | R1 |
| FR-LOG-01 | Five configurable log categories and custom logging middleware (§6). | R1 |
| FR-I18N-01 | English + Bengali: backend messages, validation, emails/SMS; frontend whole screen incl. page titles, numerals, dates, pagination text. | R1 |
| FR-PWA-01 | PWA: `manifest.json`, `service-worker.js`, install prompt, offline-first. | R4 |
| FR-OFF-01 | Offline CRUD cache in IndexedDB + outbox; auto-sync when backend reachable; conflicts surfaced. | R4 |
| FR-ASST-01 | Assistant (text + voice), deterministic, tenant-safe, permission-safe (§7). | R4 |
| FR-DOC-01 | Scalar OpenAPI docs with custom colourful theme; every endpoint titled/described. | R1 |
| FR-DOC-02 | Postman collection + environments, pre-request & test scripts, per role (Super Admin/Admin/Manager/Member). | R2 |
| FR-DEV-01 | Docker (multi-stage), compose with LB, CI/CD pipelines. | R5 (skeleton R1) |
| FR-TEST-01 | Separate projects: unit, integration, performance, load, stress (+ FE unit/e2e). | R1 → R5 |

## 5. Business rules (from the original requirement, verbatim intent)

**BR-EXP** — amount > 0; date ≤ today; category required; duplicate ⇒ warn only; members edit/delete own current-year expenses only; corrections via Adjustment only.
**BR-BUD** — one budget per category/year/tenant; none ⇒ Unlimited; only Admin sets; only current year editable; decimals OK. Soft: save + "Over Budget" + alert. Hard: held until Manager/Admin approves; approved entries record approver+time; rejected entries discarded and member told why.
**BR-CAT** — unique case-insensitive per tenant; no blank/whitespace; ≤ 50 chars; only Admin manages; deactivate hides from entry; reactivation allowed; Starter ≤ 10.
**BR-ORG** — tenant data private; only Admin invites/changes roles/removes; removed members' expenses stay; multi-org membership with selector; invitations expire in **48 h**, Admin can resend.
**BR-NOT** — in-app default on; email/SMS only Pro/Enterprise; same event once per period; approval requests immediate to all Managers/Admins.
**BR-PLAN** — see FR-PLAN-02.

## 6. Logging (five categories)

Path pattern: `logs/<category>/dd-MM-yyyy.txt`

| Category | Folder | Contains |
|---|---|---|
| Application | `application-logs` | Request lifecycle, startup/shutdown, business events (info/debug/warn) |
| Error | `application-error-logs` | Handled failures: validation/business-rule failures, dependency down, 4xx/5xx results |
| Exception | `application-exceptions-logs` | Unhandled/thrown exceptions: type, message, stack, class, file, method, line, **reason + possible fixes** |
| Query | `application-query-logs` | Every DB command: class, file, method, line, SQL, start, end, duration, rows; **slow ones flagged red** past threshold |
| Audit | `application-audit-logs` | Who/what/when/where: user, tenant, **client IP**, action, entity, correlation ID |

Rules: each category toggled independently from `appsettings.json` or env (`OUTLAY_LOGGING__...`); **never** log passwords, tokens, OTPs, secrets, connection strings or **money amounts** (redacted as `***`). Full design in `TRD.md §9`.

## 7. Assistant guardrails (summary)

Rule-based intents only; executes the **same MediatR queries and permission checks** as the UI; tenant id comes from the token, never from the question; asking about other tenants, Super Admins, or users above your level gets a neutral refusal that does not confirm existence. Full design in `TRD.md §13`.

## 8. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-01 | Monthly summary latency (warm) | ≤ 1 s at 10k orgs via pre-aggregation + cache; ≤ 3–5 s cold at scale |
| NFR-02 | API p95 for simple reads | ≤ 200 ms (cache hit) |
| NFR-03 | Availability of read paths when Redis down | 100% (fallback cache) |
| NFR-04 | Upgrade path | .NET 10 → 11+ by changing TFM/packages only; Next.js/React upgrade by dependency bump (no framework forks) |
| NFR-05 | Code quality | SOLID, DRY, KISS, YAGNI, OOP/OOAD; architecture tests enforce layering |
| NFR-06 | Security | OWASP ASVS L2 checklist; dependency scanning in CI |
| NFR-07 | Accessibility | WCAG 2.2 AA for core flows |
| NFR-08 | Test coverage | ≥ 80% domain/application lines; tenant-isolation tests mandatory |
| NFR-09 | Observability | Correlation ID end-to-end; structured logs; health endpoints |

## 9. Decisions & assumptions (confirm or change)

| ID | Decision / assumption | Why |
|---|---|---|
| A-01 | A **budget row = (tenant, category, year) holding a *monthly* allowance**. The year screen sets it once; evaluation, 80% alert and reports are monthly. | The brief says "yearly budget" but every example (৳30,000, alerts "once per month") is monthly. |
| A-02 | Hard Limit is a **Pro/Enterprise** feature (approval workflow). Starter orgs can only choose Soft. | Plan table lists "Manager approval workflow" under Pro. |
| A-03 | Lockout is manual-unlock by default; optional auto-unlock minutes. | Brief says "locked"; auto-unlock is a safe config knob. |
| A-04 | "Free SMS gateway" is a pluggable adapter (default candidate: self-hosted **TextBee** Android-phone gateway, or **Textbelt** free tier). Free tiers change; verify before relying on them. Twilio is the production path. | No free gateway is both reliable and available in Bangladesh; adapter keeps this swappable. |
| A-05 | Login is first-party: credentials → (MFA) → one-time **login ticket** → OpenIddict `/connect/token`. Auth Code + PKCE is the documented upgrade path (ADR-0003). | Supports the backend-only `sessionMinutes` field and the multi-step MFA/workspace flow with minimum moving parts. |
| A-06 | MS Access runs only on **Windows hosts** via ODBC/OLE DB and **Dapper only** (no mainstream EF Core provider). Not available inside Linux containers. | Platform limitation of Jet/ACE drivers. |
| A-07 | Running **multiple API replicas behind the load balancer requires a shared DB** (not InMemory) and Redis. InMemory ⇒ single replica. | InMemory state is per process. |
| A-08 | MediatR newer majors have **commercial licensing terms**; pin the version we are licensed for (12.x is Apache-2.0) and keep handlers behind our own `ICommand/IQuery` markers so the mediator can be swapped. **Verify current terms before upgrading.** | Licence change in 2025. |
| A-09 | Rejected hard-limit expenses are removed from the ledger; an immutable `ApprovalRequests` row (snapshot, reason, actor) is retained for audit. | Honours "permanently discarded" without losing accountability. |
| A-10 | Assistant is rule-based (deterministic). An LLM adapter may be added later behind `IAssistantEngine`, off by default. | Brief: "keep it deterministic". |
| A-11 | Source repo `github.com/mtabassum/Outlay` could not be read when these docs were written (private or empty). Treat the repo as empty/unknown until `AI-HANDOVER.md` step 1 verifies it. | Fetch failed. |

## 10. Open questions

| ID | Question | Default if unanswered |
|---|---|---|
| Q-01 | Should Managers be able to invite Members (spec says Admin only)? | No — Admin only. |
| Q-02 | Real payment gateway for Pro billing? | Out of scope for 1.0; plan switch only. |
| Q-03 | Which Bangladeshi SMS vendor for production? | Twilio adapter + generic HTTP adapter. |
| Q-04 | Hosting target for production? | Docker compose on a VPS; Kubernetes manifests later. |

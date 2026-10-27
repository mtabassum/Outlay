# Outlay — Expense Tracking & Budget Control

Multi-tenant SaaS: organisations sign up, invite their team, set category budgets and track expenses in a private workspace — in **English or বাংলা**, online or offline.

> Status: **documentation baseline (0.0.1)**. Sections marked 🔧 describe behaviour once the milestone in `Plan.md` is implemented.

## 1. Documentation map
Start with `MASTER-SPECIFICATION.md`, then `PRD.md → AppFlow.md → UiSpec.md → TRD.md → Schema.md → Plan.md`. AI agents: `CLAUDE.md`, `AI-HANDOVER.md`, `MASTER-PROMPT.md`. Developers: `docs/BACKEND-DEVELOPER-GUIDE.md`, `docs/FRONTEND-DEVELOPER-GUIDE.md`, `docs/adr/`, `docs/architecture/C4-DIAGRAMS.md`.

## 2. Quick start 🔧
```bash
git clone https://github.com/mtabassum/Outlay.git && cd Outlay
cp .env.example .env
docker compose up --build        # InMemory DB, single API replica, web on :3000, API on :8080
```
* Web: http://localhost:3000 · API docs (Scalar): http://localhost:8080/scalar · Health: `/health/ready`
* Without Docker: `dotnet run --project src/Outlay.Api` and `cd web && pnpm i && pnpm dev`.

## 3. Demo accounts 🔧 (Development only — seeded when `SeedDemoData=true`)
| Role | Email | Password |
|---|---|---|
| Super Admin | superadmin@outlay.local | `Outlay@12345` |
| Admin (TechCorp BD, Pro) | rahim@techcorp.com.bd | `Outlay@12345` |
| Manager | nusrat@techcorp.com.bd | `Outlay@12345` |
| Member | fahmida@techcorp.com.bd | `Outlay@12345` |
| Admin (Medicore Ltd, Starter 3/3) | admin@medicore.example | `Outlay@12345` |

**OTP in Development/Testing is always `111111`.** In other environments a random 6-digit code is sent by SMS and email and expires in 5 minutes. Change/disable all demo credentials before any real deployment.

## 4. How to use (end users)
1. **Sign up**: *Create Workspace* → organisation, email, password → onboarding wizard (organisation details → plan → invite team).
2. **Categories** (Admin): add names (max 50 characters, unique). **Budgets** (Admin): set the monthly amount per category for the current year; choose *Soft* (warn) or *Hard* (needs approval — Pro/Enterprise).
3. **Add expense** (everyone): date, category, amount, description. The *Budget impact* box tells you what will happen. Offline? It saves locally as *Queued* and syncs automatically.
4. **Approvals** (Manager/Admin): Notifications → *Review & Approve* → approve (optional note) or reject (reason required).
5. **Reports** (Manager/Admin): Monthly Summary by category or by member; export.
6. **Corrections**: never edit an old expense — use *Add adjustment*.
7. **Search any list**: *Add filter* → field, operator (contains, starts with, ends with, not contains, =, ≠, >, <, between, in…), value; combine with AND/OR; save the filter.
8. **Import many rows**: *Import* → download template → fill CSV/Excel → upload. If anything is wrong **nothing is saved** and you get every error with row and column plus an error file; fix and upload again.
9. **Permissions** (Admin/Super Admin): *Permissions* page → pick a person → tick what they may do (you can only give what you have).
10. **Assistant**: open it, type or press the mic — e.g. "How much did we spend on travel this month?" It only answers within what *you* are allowed to see.
11. **Language**: EN | বাংলা switch in the top bar changes the whole screen.
12. **Install as an app**: accept the *Install Outlay* prompt (iOS: Share → Add to Home Screen).

## 5. Roles at a glance
| | Super Admin | Admin | Manager | Member |
|---|---|---|---|---|
| Organisations & plans | ✔ | – | – | – |
| Tenant expense data | ✘ | ✔ | ✔ team | own only |
| Invite/remove users, budgets, categories | – | ✔ | – | – |
| Approve over-budget | – | ✔ | ✔ | – |
| Delegate permissions | to Admins | to Manager/Member (≤ own) | – | – |
| Backup/logs download | ✔ | – | – | – |

## 6. Configuration 🔧
Everything is in `appsettings.json` and overridable by environment variables with the `OUTLAY_` prefix (`__` for nesting).

| Area | Example env | Notes |
|---|---|---|
| Database | `OUTLAY_DATABASE__PROVIDER=PostgreSql` `OUTLAY_DATABASE__ORM=Dapper` `OUTLAY_DATABASE__CONNECTIONSTRING=…` | InMemory · SqlServer · MySql · Oracle · PostgreSql · MsAccess · MongoDb |
| Cache | `OUTLAY_CACHE__PROVIDER=Redis` `OUTLAY_CACHE__REDIS__CONNECTIONSTRING=redis:6379` | Falls back automatically |
| Logging | `OUTLAY_LOGGING__ENABLED=true` `OUTLAY_LOGGING__CATEGORIES__QUERY__SLOWQUERYTHRESHOLDMS=200` | Per-category switches; files in `logs/<category>/dd-MM-yyyy.txt` |
| Security | `OUTLAY_SECURITY__BCRYPT__WORKFACTOR=12` `OUTLAY_SECURITY__SESSION__MAXMINUTES=1440` | |
| MFA/SMS | `OUTLAY_MFA__ENABLED=true` `OUTLAY_SMS__PROVIDER=Twilio|FreeGateway|Mock` `OUTLAY_SMS__TWILIO__ACCOUNTSID=…` | Dev OTP 111111 |
| Resilience | `OUTLAY_RESILIENCE__RETRY__MAXATTEMPTS=3` | |
| Support contact | `OUTLAY_SUPPORT__NAME` `OUTLAY_SUPPORT__PHONE` `OUTLAY_SUPPORT__EMAIL` | Shown when a dependency is down |
| Web | `NEXT_PUBLIC_API_BASE_URL` `NEXT_PUBLIC_SUPPORT_*` | |

## 7. Operations 🔧 (Super Admin)
* Download database backup of the **active** provider: `GET /api/v1/ops/backup`.
* Download logs: `GET /api/v1/ops/logs?category=query&date=28-10-2026` (`application|error|exception|query|audit`).
* Latest release notes for QA: `GET /api/v1/release-notes/latest`.
* Health: `/health/live`, `/health/ready`.

## 8. Troubleshooting
| Symptom | Likely cause / fix |
|---|---|
| "Something went wrong, contact Outlay support" | A dependency (DB/Redis/SMS) is down — check `/health/ready`, then `logs/application-error-logs/` and `application-exceptions-logs/` (they list reason + possible fixes) |
| Data differs between API replicas | InMemory DB with >1 replica — use a real DB (A-07) |
| OTP not received | Check `Sms:Provider`; dev uses `111111`; see error log |
| Account locked | 3 wrong passwords — reset password or ask Admin to unlock |
| Slow page | Open `application-query-logs/`; red/`[SLOW]` entries show SQL, method and line |

## 9. Contributing
Read `CLAUDE.md` and the developer guides. Small PRs, tests included, conventional commits, update `RELEASENOTES.md`.

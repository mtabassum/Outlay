# Outlay — Expense Tracking & Budget Control

A multi-tenant SaaS for organisations. Each organisation signs up, invites its team, sets a budget per category and tracks expenses in its own private workspace. It works in English or বাংলা, and online or offline.

| Field | Value |
|---|---|
| Version | 0.0.0 · 2026-10-28 |
| Status | Specification baseline. No application code yet. |
| Backend | .NET 10 · ASP.NET Core Minimal APIs · EF Core 10 · MediatR · OpenIddict |
| Frontend | React 19 · TypeScript · Vite · Tailwind v4 · PWA |

> Sections marked 🔧 describe behaviour that exists once the matching milestone in [docs/Plan.md](docs/Plan.md) is built. Milestone M13-04 completes this README.

---

## 1. Documentation map

| If you are… | Start with |
|---|---|
| New to the project | [MASTER-SPECIFICATION.md](MASTER-SPECIFICATION.md), then [PRD](docs/PRD.md) → [AppFlow](docs/AppFlow.md) → [UiSpec](docs/UiSpec.md) → [TRD](docs/TRD.md) → [Schema](docs/Schema.md) → [Plan](docs/Plan.md) |
| A backend developer | [docs/guides/BACKEND-DEVELOPER-GUIDE.md](docs/guides/BACKEND-DEVELOPER-GUIDE.md) |
| A frontend developer | [docs/guides/FRONTEND-DEVELOPER-GUIDE.md](docs/guides/FRONTEND-DEVELOPER-GUIDE.md) |
| Looking for architecture pictures | [docs/diagrams/C4-DIAGRAMS.md](docs/diagrams/C4-DIAGRAMS.md) |
| Asking "why was this decided?" | [docs/adr/](docs/adr/README.md) |
| An AI agent | [CLAUDE.md](CLAUDE.md), [AI-HANDOVER.md](AI-HANDOVER.md), [MASTER-PROMPT.md](MASTER-PROMPT.md) |
| Planning or testing a release | [ROADMAP.md](ROADMAP.md), [RELEASENOTES.md](RELEASENOTES.md) |

## 2. Quick start 🔧

### 2.1 Without Docker (no database needed)

```bash
git clone https://github.com/mtabassum/Outlay.git && cd Outlay

# API
dotnet restore Outlay.slnx
dotnet run --project src/Backend/Outlay.Api --launch-profile https

# Web (second terminal)
cd src/Frontend/outlay-web
npm ci && npm run dev
```

| What | Default address | Where the default comes from / how to change it |
|---|---|---|
| API | `https://localhost:7024` (`http://localhost:5071`) | `src/Backend/Outlay.Api/Properties/launchSettings.json`. Override with `--urls "https://localhost:<port>"` or `ASPNETCORE_URLS`. Don't edit launchSettings.json. |
| API reference (Scalar) | `https://localhost:7024/scalar` | Follows the API port |
| Web app | `http://localhost:5173` | `VITE_DEV_PORT`. Point it at your API with `VITE_API_PROXY_TARGET`. Put personal overrides in `.env.development.local`. |
| Health | `/health/live`, `/health/ready` | Follows the API port |

In Development the app uses the InMemory database and seeds demo data. The OTP is always `111111`, and SMS and email go to the console.

### 2.2 Full stack with Docker

```bash
cp deploy/.env.example deploy/.env        # set passwords; change host ports here if any are taken
docker compose -f deploy/docker-compose.yml --profile postgres up -d --scale api=2
```

| Service | Default host port | Set by (`deploy/.env`) |
|---|---|---|
| Load balancer (web + API) | `443` / `80` → `https://localhost` | `LB_HTTPS_PORT`, `LB_HTTP_PORT` |
| Mailpit (emails) | `8025` → `http://localhost:8025` | `MAILPIT_UI_PORT` |
| PostgreSQL | `5432` | `POSTGRES_PORT` |
| Redis | `6379` | `REDIS_PORT` |

Other databases have their own profiles: `sqlserver`, `mysql`, `oracle`, `mongo`. Run `docker compose -f deploy/docker-compose.yml ps` to see the actual port mappings. The InMemory provider supports **one API instance only**, so use a real database before scaling `api` beyond 1.

## 3. Demo accounts 🔧

These exist only in Development (`Database:SeedDemoData=true`). A startup guard refuses to use them in any other environment.

| Role | Email | Password |
|---|---|---|
| Super Admin | `superadmin@outlay.app` | `Outlay@Dev2026!` |
| Admin (TechCorp BD, Pro) | `rahim@techcorp.com.bd` | `Outlay@Dev2026!` |
| Manager | `nusrat@techcorp.com.bd` | `Outlay@Dev2026!` |
| Member | `fahmida@techcorp.com.bd` | `Outlay@Dev2026!` |

- The OTP in Development is always **`111111`**. In other environments a random 6-digit code is sent by SMS and email. It expires in 5 minutes, and the screen shows a countdown.
- More seeded organisations and users are listed in [Schema.md §6](docs/Schema.md).

## 4. How to use (end users) 🔧

- **Sign up:** choose *Create workspace* and enter your organisation, email and password. The onboarding wizard then asks for organisation details, a plan and your team invitations.
- **Categories (Admin):** add a category name. It must be unique and 50 characters or fewer. Deactivated categories disappear from entry screens but stay in reports.
- **Budgets (Admin):** set an amount per category for the current year. That amount is the **allowance for every month** of the year. Choose **Soft** (warn) or **Hard** (needs approval; Pro and Enterprise only). A category with no budget is Unlimited.
- **Add an expense (everyone):** enter the date, category, amount and description. The *Budget impact* box shows what will happen before you save. If you're offline, the expense is saved on your device as *Queued* and syncs automatically when you're back online.
- **Edit or delete an expense:**
  - You can edit or delete only **your own** expenses.
  - The expense must be **Saved** or **Pending approval**, and its month must still be open: until the 1st of the next month plus 3 grace days.
  - Every edit re-runs all the checks, so an edit can send an expense for approval or bring it back within budget.
  - Approved and Rejected expenses, and closed months, are locked. Correct those with **Add adjustment**.
- **Approvals (Admin, and Managers with approval rights):** open Notifications → *Review & Approve*. Approve with an optional note, or reject with a reason (required). You only see requests for members you're allowed to approve.
- **Over budget** is shown as "Over by ৳…". Negative amounts never appear.
- **Reports (Manager/Admin):** *Monthly Summary* by category or by member. Export is available on Pro and Enterprise.
- **Search any list:** choose *Add filter*, pick a field, an operator (equals, contains, starts with, ends with, not contains, >, <, between, in…) and a value. Combine conditions with AND/OR, and save the filter for later.
- **Import many rows:** choose *Import*, download the template, fill in the CSV or Excel file and upload it. If any row is wrong, **nothing is saved**. You get every error with its row and column, plus an annotated file to fix and upload again.
- **Permissions (Super Admin/Admin):** open the Permissions page, pick a person and tick what they may do. You can only give permissions you have yourself. Removing a permission also removes it from everyone who received it through that person.
- **Assistant:** type or press the mic and ask, for example, "How much did we spend on travel this month?" It answers only from data you're allowed to see.
- **Language:** the **EN | বাংলা** switch in the top bar changes the whole screen, including numbers and dates.
- **Install as an app:** accept the *Install Outlay* prompt. On iOS, use Share → Add to Home Screen.

## 5. Roles at a glance

| | Super Admin | Admin | Manager | Member |
|---|---|---|---|---|
| Organisations and plans | ✔ | – | – | – |
| Tenant expense data | ✘ never | ✔ all | ✔ whole organisation by default (an Admin can narrow it to a team) | Own only |
| Invite/remove users, categories, budgets | – | ✔ | – (unless delegated) | – |
| Approve over-budget expenses | – | ✔ | ✔ if granted `expenses.approve`, within their scope | – |
| Delegate permissions | To Admins | To Managers and Members (only from their own permissions) | – | – |
| Backups and log download | ✔ | – | – | – |

Full rules: [PRD §3](docs/PRD.md#3-personas--roles).

## 6. Configuration 🔧

Settings live in `appsettings.json`. Any setting can be overridden by an environment variable or a line in `deploy/.env`, using `__` for nesting. Never commit secrets to `appsettings*.json`.

| Area | Example | Notes |
|---|---|---|
| Database | `Database__Provider=PostgreSql` | `InMemory` · `SqlServer` · `PostgreSql` · `MySql` · `Oracle` · `Access` · `MongoDb` ([TRD §6.2](docs/TRD.md)) |
| Cache | `Cache__L2=Redis` | `Redis` · `Memcached` · `None` |
| Logging | `OutlayLogging__Categories__Query__SlowQueryThresholdMs=500` | Each category can be switched on or off. Files are written to `logs/<category>/dd-MM-yyyy.txt`. |
| OTP / SMS | `Otp__UseFixedDevelopmentCode=false` · `Sms__Provider=Twilio` | `Console` · `TextBelt` · `Twilio` |
| Lockout | `Auth__Lockout__MaxFailedAttempts=3` | Lockout duration is also configurable |
| Support contact | `Support__TeamName`, `Support__Phone`, `Support__Email` | Shown whenever a dependency is down |
| Web | `VITE_API_BASE_URL=/api` · `VITE_SUPPORT_NAME` / `_PHONE` / `_EMAIL` | The API base URL is relative, so it works on any port |

Full reference: [CLAUDE.md §10](CLAUDE.md) and [TRD §17](docs/TRD.md).

## 7. Operations 🔧 (Super Admin)

| Need | Endpoint |
|---|---|
| Back up the active database | `POST /api/v1/system/backups`, then `GET /api/v1/system/backups/{id}/download` (step-up OTP required) |
| Download logs | `GET /api/v1/system/logs/{category}/{date}`, for example `/query/28-10-2026`. Categories: `application`, `error`, `exception`, `query`, `audit` |
| Release notes for SQA | `GET /api/v1/system/release-notes/latest` (source: [RELEASENOTES.md](RELEASENOTES.md)) |
| Version and health | `GET /api/v1/system/version` · `/health/live` · `/health/ready` |

Postman collections for each role are in `postman/`.

## 8. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| "Something went wrong… contact the Outlay team" | A dependency (database, Redis, SMS) is down. Check `/health/ready`, then `logs/application-error-logs/` and `logs/application-exceptions-logs/`, which record the probable cause and suggested fixes. Quote the reference shown on screen. |
| Data differs between API replicas | You're running the InMemory database with more than one replica. Use a real database. |
| Port already in use | Override it as described in §2 (`--urls`, `.env.development.local` or `deploy/.env`). |
| OTP not received | In Development the OTP is `111111`. Otherwise check `Sms:Provider` and the error log; emails appear in Mailpit. |
| Account locked | Three wrong passwords lock the account. Reset your password, wait for the lockout to expire, or ask an Admin to unlock you. |
| Slow page | Open `logs/application-query-logs/`. Lines marked `[SLOW QUERY ⚠ …]` show the SQL, the calling method and the line number. |
| Edit refused with "month closed" | The expense's month is closed. Use **Add adjustment** instead. |

## 9. Contributing

- Read [CLAUDE.md](CLAUDE.md) and the developer guide for your side.
- Keep PRs small and include tests. The Definition of Done is in [docs/Plan.md](docs/Plan.md).
- Use Conventional Commits with the task ID, for example `feat(expenses): M2-04 create expense slice`. Use the `docs:` prefix for documentation-only changes.
- Update `RELEASENOTES.md` (Unreleased section plus the SQA checklist) and `AI-HANDOVER.md`.

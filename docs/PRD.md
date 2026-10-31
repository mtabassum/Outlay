# Outlay — Product Requirements Document (PRD)

| Field | Value |
|---|---|
| Product | **Outlay** — Multi-tenant Expense Tracking & Budget Control SaaS |
| Document | PRD.md (1 of 6 core specs) |
| Version | 0.0.0 (specification baseline) |
| Date | 2026-10-27 |
| Owner | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |
| Source requirements | `Requirement.pdf` — *Expense Tracking & Budget Control System — Multi-Tenant SaaS Platform (OOP Training Program, Project 3)* + extended engineering requirements from the project owner |
| Related | [AppFlow.md](AppFlow.md) · [UiSpec.md](UiSpec.md) · [TRD.md](TRD.md) · [Schema.md](Schema.md) · [Plan.md](Plan.md) · [../MASTER-SPECIFICATION.md](../MASTER-SPECIFICATION.md) |

> **How to read this document.** The PRD says *what* Outlay must do and *why*. It doesn't say *how*; that's in TRD.md and Schema.md.
> Every requirement has a stable ID (`FR-*`, `BR-*`, `NFR-*`, `US-*`). Code, tests and commits should reference these IDs.

---

## 1. Vision & Purpose

Companies need a simple way to track employee expenses, set budgets for each expense category, and get alerted before they overspend.
Outlay serves **many organisations at once**. Each organisation gets its own private workspace, and only its own team can see its data.

**Vision statement:** *"Every taka accounted for, before it's spent."* Outlay should feel faster than a spreadsheet, safer than a shared drive, and keep working when the network doesn't.

### 1.1 Goals

| # | Goal | Measure of success |
|---|---|---|
| G1 | Organisations can self-onboard and start tracking expenses in under 10 minutes | Median time from "Create Workspace" to first expense < 10 min |
| G2 | Overspending is caught *before* month end | 100% of 80% threshold crossings produce exactly one alert per category per month |
| G3 | Absolute data isolation between tenants | 0 cross-tenant data leaks (automated isolation tests in CI) |
| G4 | Works offline and syncs reliably | ≥ 99.5% of offline-queued mutations sync without manual intervention |
| G5 | Fast at scale (US-20) | Monthly Summary p95 < 1 s at 1M expenses/tenant-month on reference hardware (requirement target: 3–5 s) |
| G6 | Fully bilingual | 100% of UI strings, numbers, dates and pagination localised in English and Bengali (বাংলা) |
| G7 | Production-grade operations | Categorised logs, slow-query detection, backups via API, release notes via API, CI/CD, Docker, load-tested |

### 1.2 Non-Goals (v1)

- Payment collection / subscription billing integration (plan changes are performed by Super Admin or a stubbed "upgrade request" flow; Stripe/SSLCommerz are on the roadmap).
- Receipt OCR, mileage tracking, corporate card feeds.
- Accounting-system export (QuickBooks, Tally).
- A generative-AI (LLM) assistant. The v1 assistant is **deterministic** by design (see FR-AST).
- Native mobile apps. The PWA covers mobile.

---

## 2. Multi-Tenancy

A single application instance serves many organisations at the same time. Each organisation's expenses, categories, budgets and users are **completely private**.

> **Example:** TechCorp BD and Medicore Ltd both use Outlay. TechCorp's employees can only see TechCorp's expenses. Medicore's team sees only Medicore's. The system keeps them separate automatically.

A single person (one login, one email) **may belong to several organisations with different roles in each** (e.g., Manager at TechCorp BD and Member at another company). After login they see a **workspace selector**.

---

## 3. Personas & Roles

### 3.1 Personas (from the business scenario)

| Persona | Organisation | Role | Daily use |
|---|---|---|---|
| **Rahim Ahmed** | TechCorp BD (Pro plan, 8 employees) | Tenant Admin | Monitors budget health on the dashboard, manages team members, handles billing |
| **Nusrat Jahan** | TechCorp BD | Manager | Reviews team expenses, approves over-budget entries, views monthly report |
| **Fahmida Akter, Tarek, Karim…** | TechCorp BD | Member (User) | Adds their own expenses; can only see their own history |
| **Platform Operator** | — (Outlay team) | Super Admin | Manages organisations, subscription plans, platform health, logs, backups |
| **SQA Engineer** | Outlay team | (uses Release Notes API, Postman suites) | Verifies each deployment against release notes |

### 3.2 Roles (four built-in roles)

| Role | Who they are | What they can do (default) | What they cannot do |
|---|---|---|---|
| **Super Admin** | Platform operator (not part of any organisation) | View all organisations, manage subscription plans, monitor platform health, create Tenant Admins, grant permissions to Admins, read logs, download DB backups | Cannot see any organisation's expense data (amounts, descriptions, members' spending) |
| **Admin** (Tenant Admin) | Organisation owner or IT lead | Invite & remove users, set roles, create categories, set budgets, view all expenses, manage billing, create Managers & Users and delegate permissions to them | Cannot access other organisations' workspaces; cannot grant a permission they don't hold |
| **Manager** | Team lead or finance officer | View all team expenses (default scope: whole organisation, can be narrowed to own team), approve over-budget entries, view monthly report, add own expenses | Cannot manage users or change billing plan (unless an Admin delegates those permissions) |
| **User** (Member) | Regular employee | Add, edit and view **own** expenses only | Cannot see other members' expenses or reports |

> **Naming:** the requirement says *Tenant Admin* and *Member*. The owner's brief says *Admin* and *User*. They mean the same roles. The UI uses **Admin** and **Member**, and the code uses `TenantAdmin` and `Member`. See the Glossary in MASTER-SPECIFICATION.md.

### 3.3 Delegated Permission Model (owner requirement)

Roles are **templates** that pre-fill a set of permissions. The actual authorisation unit is the **permission**.

1. **Super Admin** creates an **Admin** for an organisation and grants a subset of features (permissions).
2. An **Admin** sees and uses **only** the permissions granted to them.
3. An **Admin** creates **Managers** and grants them permissions **only from the Admin's own set**.
4. An **Admin** (or anyone holding `users.create`) creates **Users** and grants permissions only from their own set.
5. **Revoking** a permission from someone **cascades** to everyone who received it through them (after a confirmation dialog that lists the affected people).
6. There is a **dedicated Permissions page** in the UI and a **dedicated Permissions API**.

---

## 4. Subscription Plans

| | **Starter** (Free) | **Pro** (Most popular) | **Enterprise** |
|---|---|---|---|
| Price | $0 forever | $49 / month | Custom pricing |
| Users | Up to **3** | Up to **15** | Unlimited |
| Categories | Up to **10** | Unlimited | Unlimited |
| Expense history visible | **6 months** | Full | Full + archive export |
| Reports | Basic monthly report | Advanced reports & member filter | Custom reports & data export |
| Notifications | In-app only | In-app + Email & SMS alerts | In-app + Email & SMS |
| Hard Limit & Manager approval workflow | — (Soft Limit only) | ✔ | ✔ |
| Single sign-on (SSO) | — | — | ✔ (roadmap v1.2) |
| Support | Community | Standard | Priority |

### 4.1 Plan limits enforced by the system (BR-PLAN)

- **BR-PLAN-01** Starter with 3 users → inviting a 4th is **blocked** with an upgrade prompt.
- **BR-PLAN-02** Starter trying to create an 11th category → **blocked**, upgrade suggested.
- **BR-PLAN-03** Starter cannot see expense history older than 6 months. The data is **kept**, just hidden until upgrade.
- **BR-PLAN-04** An organisation cannot downgrade to a plan whose limits its current usage exceeds.
- **BR-PLAN-05** Email/SMS *alerts* are available on Pro and Enterprise only. Authentication OTP by SMS/Email is a security feature and is available on **all** plans.
- **BR-PLAN-06** Hard Limit budgets and the approval workflow require Pro or Enterprise.

---

## 5. Functional Requirements

Priority: **P0** = MVP (requirement's "Build These First"), **P1** = v1.0 GA, **P2** = post-GA.

### 5.1 Organisation & Workspace (FR-ORG)

| ID | Requirement | Priority |
|---|---|---|
| FR-ORG-01 | Organisation sign-up creates a private workspace (name, owner email, password) | P0 |
| FR-ORG-02 | Owner chooses a plan (Starter / Pro / Enterprise "Contact sales") | P0 |
| FR-ORG-03 | Owner automatically becomes Tenant Admin | P0 |
| FR-ORG-04 | 4-step onboarding wizard: ✔ Create Account → Set Up Organisation (name, industry, team size, currency) → Choose Plan → Invite Team; later prompts guide: Create categories → Set budgets → Invite team | P0 |
| FR-ORG-05 | Login with organisation context; multi-membership users see a workspace selector | P0 |
| FR-ORG-06 | Super Admin can create an organisation and its Admin directly | P1 |
| FR-ORG-07 | Super Admin can suspend/reactivate an organisation and change its plan | P1 |

### 5.2 Identity, Security & Sessions (FR-AUTH)

| ID | Requirement | Priority |
|---|---|---|
| FR-AUTH-01 | OAuth 2.0 / OpenID Connect authorisation server (OpenIddict) on top of ASP.NET Core Identity | P0 |
| FR-AUTH-02 | Passwords hashed with **BCrypt** (work factor configurable, default 12). Passwords are never stored reversibly. | P0 |
| FR-AUTH-03 | **3 consecutive wrong passwords lock the account** (lockout duration configurable). Unlock by password reset, lockout expiry, or Admin unlock. | P0 |
| FR-AUTH-04 | Forgot password (email link + OTP) and Change password (requires current password + OTP) | P0 |
| FR-AUTH-05 | **MFA / 2FA** by one-time password (OTP) sent by **SMS and Email**. OTP valid **5 minutes**, with a visible countdown. Resend allowed after 60 s. Max 5 attempts per code. | P0 |
| FR-AUTH-06 | **Development** environment: OTP is always **`111111`** and nothing is sent. Any other environment generates a cryptographically random 6-digit OTP and sends it. The app refuses to start if the fixed OTP is enabled outside Development. | P0 |
| FR-AUTH-07 | SMS providers are pluggable and configured in appsettings/.env: `Console` (dev), `TextBelt` (free tier, testing only), `Twilio` (production). | P0 |
| FR-AUTH-08 | Short-lived access tokens + **refresh-token rotation** (each refresh token is single-use) + **revocation** (logout, password change/reset, lockout, permission change revoke tokens) | P0 |
| FR-AUTH-09 | **Optional `session_duration` parameter on the login API** lets the caller pick a manual session length (e.g., `4h`, `1d`), bounded by a configured maximum. **The web UI never shows or sends this option.** | P1 |
| FR-AUTH-10 | Invitation acceptance: link valid 48 h; invitee sets a password (and phone for OTP) and joins | P0 |
| FR-AUTH-11 | Users can view and revoke their active sessions | P1 |
| FR-AUTH-12 | Enterprise SSO (Microsoft Entra ID / Google) | P2 |

### 5.3 Roles & Permissions (FR-PERM)

| ID | Requirement | Priority |
|---|---|---|
| FR-PERM-01 | Permission catalogue (see TRD §7.3), grouped by module | P0 |
| FR-PERM-02 | Role templates: SuperAdmin, TenantAdmin, Manager, Member with default permission sets | P0 |
| FR-PERM-03 | A grantor can only grant permissions they hold (*subset rule*) | P0 |
| FR-PERM-04 | Revocation cascades down the grant chain, with a preview of affected users | P1 |
| FR-PERM-05 | Dedicated Permissions API (list catalogue, get effective permissions, grant, revoke, preview cascade) | P0 |
| FR-PERM-06 | Dedicated Permissions UI page (hierarchy tree + permission matrix) | P1 |
| FR-PERM-07 | Data scope is part of the permission (`own`, `team`, `all` within tenant) | P0 |

### 5.4 Team Management (FR-TEAM)

| ID | Requirement | Priority |
|---|---|---|
| FR-TEAM-01 | Invite by email + role. Status: Invited → Active. Resend and revoke invitation. | P0 |
| FR-TEAM-02 | Plan seat check before invite (shows "Plan: Pro — 8 of 15 seats used") | P0 |
| FR-TEAM-03 | Change role, remove member (past expenses remain in records) | P0 |
| FR-TEAM-04 | Admin can unlock a locked member and force a password reset | P1 |

### 5.5 Expense Categories (FR-CAT)

| ID | Requirement | Priority |
|---|---|---|
| FR-CAT-01 | Create / edit / deactivate / reactivate categories (Admin) | P0 |
| FR-CAT-02 | Deactivated categories are hidden from entry screens but stay visible in reports | P0 |
| FR-CAT-03 | Uniqueness (case-insensitive) within an organisation; max 50 characters; not blank | P0 |

### 5.6 Expenses (FR-EXP)

| ID | Requirement | Priority |
|---|---|---|
| FR-EXP-01 | Add expense: date, category, amount (৳), description | P0 |
| FR-EXP-02 | Members see only their own expenses. Managers and Admins see the team per their data scope. | P0 |
| FR-EXP-03 | Exact-duplicate warning (same member, date, amount, category, description) that doesn't block. The user confirms to save. | P0 |
| FR-EXP-04 | Budget check on save → Soft Limit: saved + "Over Budget" flag + alert. Hard Limit exceeded: held as **Pending Approval**. | P0 (Soft) / P1 (Hard) |
| FR-EXP-05 | Confirmation or "Waiting for Approval" message after submit | P0 |
| FR-EXP-06 | Edit/delete own **Saved** or **Pending** expenses while their month is open. **Every edit re-runs validation, the duplicate check and the budget check**, and the status can change as a result (BR-EXP-05, BR-EXP-08…13). | P0 (Soft) / P1 (Hard-limit transitions) |
| FR-EXP-07 | Smarter (fuzzy) duplicate detection (US-21): "This looks similar to an entry you saved on 14 June" that warns and never blocks | P2 |

### 5.7 Budgets (FR-BUD)

| ID | Requirement | Priority |
|---|---|---|
| FR-BUD-01 | Yearly budget setup screen: one budget per category per year, with amount and limit type (Soft/Hard) | P0 |
| FR-BUD-02 | Categories without a budget are **Unlimited** and never alert | P0 |
| FR-BUD-03 | Previous years' budgets are locked | P0 |
| FR-BUD-04 | Budget monitoring per month: alert at 80% (once per category per month), over-budget at 100% | P0 |

> **Interpretation (ADR-0014):** the requirement's mock-ups show the same figure (e.g., ৳30,000 for Travel) as both the "yearly budget" record and the **monthly** budget in the Monthly Summary and Dashboard (Total Budget ৳1,50,000 = sum of the five categories). Outlay therefore treats the yearly budget record as the **monthly allowance for every month of that year**. Monitoring and alerts run per calendar month. Confirmed 2026-10-27 (Q-01 resolved).

### 5.8 Approvals (FR-APR)

| ID | Requirement | Priority |
|---|---|---|
| FR-APR-01 | Approval request notifications sent **immediately** to the expense's **approvers**: all Admins, plus Managers holding `expenses.approve` whose visibility covers the member (BR-NOT-04) | P1 |
| FR-APR-02 | Review & Approve screen shows the expense and current budget status | P1 |
| FR-APR-03 | Approve (optional note) → saved with approver + timestamp. Member notified. | P1 |
| FR-APR-04 | Reject (reason required) → entry discarded from all totals and lists. Member notified with the reason. | P1 |

### 5.9 Adjustments & Corrections (FR-ADJ)

| ID | Requirement | Priority |
|---|---|---|
| FR-ADJ-01 | Correction recorded as a separate **Adjustment Entry** linked to the original expense. The original is never changed. | P1 |
| FR-ADJ-02 | Adjustment has a direction (Reduction / Increase), a positive amount, and a reason. The resulting net amount can never go below zero. | P1 |
| FR-ADJ-03 | Monthly reports reflect adjustments in the month the adjustment is dated. Negative numbers never appear anywhere. | P1 |

### 5.10 Reports (FR-RPT)

| ID | Requirement | Priority |
|---|---|---|
| FR-RPT-01 | Dashboard (Admin view): Team Spent, Total Budget, Team Members (8/15), Pending Approvals, Budget Health bars, Recent Expenses, quick actions | P0 |
| FR-RPT-02 | Monthly Summary (Manager & Admin): totals, per-category Budget/Spent/Remaining/Status, filters by member and month | P0 |
| FR-RPT-03 | Member-wise spending (Manager view): total and number of entries per member | P1 (Pro+) |
| FR-RPT-04 | Export report (CSV/XLSX). Pro: monthly report export. Enterprise: full data export. | P1 |
| FR-RPT-05 | Super Admin platform overview: organisation, plan, users (n/limit), status (Active / Plan Limit Reached / Suspended), joined date. **No expense data.** | P0 |
| FR-RPT-06 | Pre-calculated daily totals and closed-month snapshots so reports stay fast at scale (US-20) | P1 |

### 5.11 Notifications (FR-NOT)

| ID | Requirement | Priority |
|---|---|---|
| FR-NOT-01 | In-app notifications on by default for all users. Real-time push (SignalR) plus a notifications screen. | P0 |
| FR-NOT-02 | Types: Approval Required, Over Budget, 80% Alert, New Member Joined, Seats Usage, Approval Decision, Plan Limit Reached | P0/P1 |
| FR-NOT-03 | Same-event de-duplication: one notification per event per period | P0 |
| FR-NOT-04 | Email and SMS alerts for Pro/Enterprise | P1 |
| FR-NOT-05 | Mark one / all as read | P0 |

### 5.12 Advanced Search on every CRUD (FR-SRCH)

| ID | Requirement | Priority |
|---|---|---|
| FR-SRCH-01 | Every CRUD list supports **multi-field advanced search** in the style of Jira/GitLab: conditions combined with AND/OR groups | P1 |
| FR-SRCH-02 | Operators: `equals`, `not equals`, `contains`, `not contains`, `starts with`, `not starts with`, `ends with`, `not ends with`, `>`, `≥`, `<`, `≤`, `between`, `in`, `not in`, `is empty`, `is not empty` | P1 |
| FR-SRCH-03 | Quick free-text search box + visual filter builder + text query language (OQL, e.g. `category = "Travel" AND amount >= 1000 AND description ~ "taxi"`) | P1 |
| FR-SRCH-04 | Sort by any whitelisted field; server-side pagination; localised pagination text | P1 |
| FR-SRCH-05 | Saved filters (personal, and shared with the team if permitted) | P2 |

### 5.13 Bulk Import (FR-IMP)

| ID | Requirement | Priority |
|---|---|---|
| FR-IMP-01 | Every CRUD resource where it makes sense (Categories, Budgets, Expenses, Team invitations, Adjustments) supports **bulk insert from CSV and Excel (.xlsx)** | P1 |
| FR-IMP-02 | Downloadable templates (English or Bengali headers) | P1 |
| FR-IMP-03 | **All-or-nothing**: if any row has a validation error or duplicate (inside the file or against existing data), **nothing is saved** and the whole import is rolled back | P1 |
| FR-IMP-04 | The error report points to **every** failing row, column and reason (not only the first), on screen and as a downloadable annotated file with error cells highlighted in red and a comment on each, so the user can fix the file and re-upload | P1 |
| FR-IMP-05 | Large files process in the background with a progress bar | P1 |

### 5.14 Offline-First & PWA (FR-OFF)

| ID | Requirement | Priority |
|---|---|---|
| FR-OFF-01 | Installable **PWA** (manifest.json + service-worker.js) with an "Install Outlay as an app" prompt | P0 |
| FR-OFF-02 | Data and CRUD operations are cached in **IndexedDB**. If the backend is unreachable, users can still browse cached data and create/edit records. | P1 |
| FR-OFF-03 | Automatic background sync when back online, with idempotency keys so no record is duplicated; conflicts and server-side rejections are shown to the user to resolve | P1 |
| FR-OFF-04 | Clear offline/online indicator and pending-sync counter | P1 |

### 5.15 Localisation (FR-I18N)

| ID | Requirement | Priority |
|---|---|---|
| FR-I18N-01 | English and **Bengali (বাংলা)** in both backend and frontend | P0 |
| FR-I18N-02 | Switching language changes **the whole screen**: labels, page title, validation messages, API error messages, **numbers in Bengali digits (০-৯)**, dates, currency, pagination ("পৃষ্ঠা ২ / ১০") | P0 |
| FR-I18N-03 | Language preference saved per user. Backend honours `Accept-Language`. | P0 |
| FR-I18N-04 | Taka amounts use South-Asian digit grouping (৳1,50,000) | P0 |

### 5.16 Voice & Text Assistant (FR-AST) — US-22

| ID | Requirement | Priority |
|---|---|---|
| FR-AST-01 | Chat panel that accepts **typed or spoken** questions (English and Bengali) and answers in text and **speech** | P1 |
| FR-AST-02 | **Deterministic**: the same question over the same data always gets the same answer. Intent-and-template engine, no generative model. | P1 |
| FR-AST-03 | Answers use exactly the same authorisation as the screens. A Member asking "what did the whole team spend?" gets the same "access denied" as opening the Monthly Summary. | P1 |
| FR-AST-04 | **Guardrails**: never discloses another organisation's data or Super Admin data. An Admin can ask about their Managers and Users. Tenant comes only from the token, never from the question text. | P1 |
| FR-AST-05 | Supported questions (minimum): top spending category in a period; highest spender this month; forecast "will we go over Meals this month?"; list expenses by category/amount/month; month-over-month comparison; budget remaining in a category; pending approvals; "how much did <member> spend today/this month"; my spending | P1 |

### 5.17 Operations APIs (FR-OPS)

| ID | Requirement | Priority |
|---|---|---|
| FR-OPS-01 | **DB backup download API** for whichever provider and connection string are active | P1 |
| FR-OPS-02 | **Log download API** by category (`application`, `error`, `exception`, `query`, `audit`) and date `dd-MM-yyyy` | P1 |
| FR-OPS-03 | **Release Notes API** that shows what is new in the deployed build and **what SQA must test** | P1 |
| FR-OPS-04 | Version/health endpoints (`/health/live`, `/health/ready`, `/api/v1/system/version`) | P0 |
| FR-OPS-05 | Support contact (team name, phone, email) configurable through appsettings/.env and shown on every error screen | P0 |

### 5.18 API Documentation & Testing Assets (FR-DOC)

| ID | Requirement | Priority |
|---|---|---|
| FR-DOC-01 | Every endpoint has a title (summary), a description, tags, request/response examples and error codes in OpenAPI | P0 |
| FR-DOC-02 | **Scalar** API reference with a colourful, branded Outlay theme | P0 |
| FR-DOC-03 | **Postman** collection + environments with pre-request and test scripts, organised by role (Super Admin, Admin, Manager, User), including RBAC negative tests | P1 |
| FR-DOC-04 | README, User Guide, Backend Developer Guide, Frontend Developer Guide ("how to build a CRUD slice"), ADRs, C4 diagrams, ER diagrams | P1 |

---

## 6. Business Rules (canonical)

### 6.1 Expense entry (BR-EXP)
- **BR-EXP-01** Amount must be > 0. Zero or negative amounts can't be saved.
- **BR-EXP-02** Expense date cannot be in the future (organisation time zone).
- **BR-EXP-03** Category is mandatory and must be active.
- **BR-EXP-04** Same date + amount + category + description (same member, case- and whitespace-insensitive) → warn, don't block.
- **BR-EXP-05** A user can edit or delete **only their own** expenses, and only when the expense is **Saved** or **Pending approval** and its **month is still open** in the **current year**. A month stays open until its snapshot is taken on the 1st of the next month + `Reports:MonthCloseGraceDays` (default 3). **Approved** and **Rejected** entries, closed months and previous years are locked, and corrections go through Adjustments. (Q-02 resolved.)
- **BR-EXP-06** Corrections and refunds are Adjustment Entries. The original expense is never modified after approval/closure.
- **BR-EXP-07** Amounts support 2 decimals. Currency is the organisation's currency (default BDT ৳).
- **BR-EXP-08** **Every edit re-runs the same checks as create:** validation (BR-EXP-01…03, and the new date must also be in an open month of the current year), the exact-duplicate warning (BR-EXP-04, **excluding the expense itself**), and the budget check for the (possibly new) category and month. The budget check counts that month's spend **without the expense's previous amount**, then adds the new amount.
- **BR-EXP-09** **The status after an edit follows the budget check:**
  - Saved → **Pending approval** when the edit exceeds a **Hard** limit. An approval request goes to the approvers (BR-NOT-04).
  - Pending approval → **Saved** when the edit brings it within budget, or into an unlimited category. The approval request is **withdrawn** and approvers are notified.
  - **Soft** limit: the "Over budget" flag is recalculated and the expense stays Saved.
  - 80% and over-budget alerts still fire at most once per category per month (BR-OVR-01, BR-NOT-03).
- **BR-EXP-10** **Delete** follows BR-EXP-05 (own, Saved/Pending, open month). It's a soft delete: the amount leaves the totals and any pending approval request is cancelled.
- **BR-EXP-11** **Grace window:** during the grace days, both the previous and the current month are open. An edit may move an expense's date between them, and both months' totals update.
- **BR-EXP-12** **Concurrency and offline:** an edit must carry the expense's current version. If it changed meanwhile (e.g. a Manager approved it), the edit is refused with 409 and the current state. An edit or delete that arrives after the month closed (including an offline change synced late) is refused with `month_closed`, and the UI offers to create an Adjustment instead.
- **BR-EXP-13** Managers and Admins never edit other people's expenses. They correct them with Adjustments (`adjustments.create.any`). Every edit and delete is audited with the changed field names.

### 6.2 Budgets (BR-BUD)
- **BR-BUD-01** Only one budget per category per year per organisation.
- **BR-BUD-02** No budget means Unlimited, and no alerts fire.
- **BR-BUD-03** Only Tenant Admin (or holder of `budgets.manage`) can set/change a budget, and only for the current year (and next year, for planning).
- **BR-BUD-04** Budget amounts may include decimals (e.g., ৳15,000.50).

### 6.3 Categories (BR-CAT)
- **BR-CAT-01** Unique within organisation, case-insensitive ("Travel" = "travel").
- **BR-CAT-02** Not blank and not whitespace-only. Max 50 characters.
- **BR-CAT-03** Only Admin (or `categories.manage`) can create, edit or deactivate.
- **BR-CAT-04** Deactivated categories are hidden on entry screens, but past expenses stay in reports. They can be reactivated at any time.
- **BR-CAT-05** Starter plan: max 10 categories.

### 6.4 Overspending (BR-OVR)
- **BR-OVR-01** At ≥ 80% of the monthly budget an alert fires **once per month per category**.
- **BR-OVR-02** Soft Limit: the expense is saved and marked "Over Budget", and an alert goes to Admins and Managers.
- **BR-OVR-03** Hard Limit: the expense can't be saved without Manager/Admin approval. It waits in "Pending" state.
- **BR-OVR-04** An approved Hard Limit entry is saved with a note of who approved it and when.
- **BR-OVR-05** A rejected Hard Limit entry is discarded: excluded from every total, list and report, and the member is told the reason. (A tombstone is kept for audit only. See Schema.md.)
- **BR-OVR-06** Pending entries are **not** counted in "Spent" until approved, but are shown as "Pending" on the Dashboard.

### 6.5 Organisation & workspace (BR-ORG)
- **BR-ORG-01** Each organisation's data is completely private.
- **BR-ORG-02** Only Tenant Admin (or delegated `users.invite` / `roles.assign`) can invite users, change roles, or remove members.
- **BR-ORG-03** A removed member's past expenses stay in the organisation's records.
- **BR-ORG-04** One person can belong to several organisations, and sees a workspace selector after login.
- **BR-ORG-05** Invitation links expire after 48 hours. After that the Admin must resend.

### 6.6 Notifications (BR-NOT)
- **BR-NOT-01** In-app notifications are on by default for all users.
- **BR-NOT-02** Email/SMS alerts on Pro and Enterprise only.
- **BR-NOT-03** The same event (e.g., the 80% alert for the same category) is sent once per period.
- **BR-NOT-04** Approval requests go immediately to the expense's **approvers**: **all Admins** of the organisation, plus **Managers who hold `expenses.approve` and whose visibility covers the member** (Q-04). The submitter is never their own approver, except that an Admin may self-approve when no other approver exists, which is flagged in the audit log. "Request withdrawn" and "request cancelled" notifications go to the same group. The decision goes to the member.
- **BR-NOT-05** The approver group is evaluated **when the queue is read and when approving**, not stored. If an Admin narrows a Manager's visibility, pending requests outside the new scope disappear from that Manager's queue and notification list, and approving them returns 404. With the default whole-organisation scope, every Manager with `expenses.approve` is an approver.

### 6.7 Security (BR-SEC)
- **BR-SEC-01** No passwords, OTPs, tokens, secrets, card numbers or **money amounts** in any log file. They are masked as `***`.
- **BR-SEC-02** Audit log records actor, tenant, action, entity, timestamp (UTC), **client IP address** (honouring trusted proxies), user agent and correlation ID.
- **BR-SEC-03** All mutating requests accept an `Idempotency-Key`, and offline sync always sends one.
- **BR-SEC-04** Every request has a correlation ID (`X-Correlation-Id`) that is returned in responses and printed in all logs.

---

## 7. User Stories (selected; acceptance criteria in Gherkin)

**US-01 Organisation sign-up (Admin)**
```gherkin
Given I am on the Login screen
When I click "Create Workspace", enter "TechCorp BD", my email and a strong password
And I complete organisation details, choose "Pro" and skip inviting
Then a private workspace "TechCorp BD" exists, I am its Admin,
And I land on the Dashboard with a "Create categories → Set budgets → Invite team" checklist
```

**US-02 Add expense within budget (Member)**
```gherkin
Given Travel has a Soft Limit budget of ৳30,000 and ৳20,000 is spent this month
When I add ৳1,200 "Taxi to client — Gulshan" dated today in Travel
Then the expense is saved with status "Saved" and I see a confirmation toast
```

**US-03 Hard limit exceeded (Member → Manager)**
```gherkin
Given Travel has a Hard Limit budget of ৳30,000 and ৳32,100 is already spent
When I submit ৳1,200 in Travel
Then I see the Budget Impact warning and the button reads "Submit for Approval"
And the expense is stored as "Pending Approval"
And my approvers (all Admins, plus Managers whose visibility covers me) get an "Approval Required" notification immediately
```

**US-04 80% alert fires once**
```gherkin
Given Meals has a ৳25,000 budget and ৳19,900 is spent in June
When an expense brings June's Meals total to ৳20,400 (≥80%)
Then one "Meals & Entertainment — 80% Alert" notification is created for Admins and Managers
And further Meals expenses in June do not create another 80% alert
```

**US-05 Bulk import with errors**
```gherkin
Given I upload expenses.xlsx with 500 rows where row 17 has amount "-50" and rows 30 and 31 are identical
When the import is validated
Then nothing is saved
And I see 3 errors: (row 17, Amount, "Amount must be greater than zero"),
    (row 31, —, "Duplicate of row 30 in this file"), (row 30, —, "Duplicate of row 31 in this file")
And I can download expenses.errors.xlsx with those cells highlighted in red and commented
```

**US-06 Offline expense**
```gherkin
Given I am logged in and the network goes offline
When I add an expense
Then it appears in my list with a "Pending sync" badge
And when the network returns it is sent with its Idempotency-Key exactly once
And the badge disappears, or shows "Needs approval" if the server holds it for approval
```

**US-07 Lockout**
```gherkin
Given my account is active
When I enter a wrong password 3 times in a row
Then my account is locked and I see "Your account is locked. Reset your password or try again in 15 minutes."
And an audit entry with my IP address is written
```

**US-08 Permission delegation**
```gherkin
Given Admin Rahim holds expenses.read.all and reports.monthly.read but not reports.export
When Rahim opens the Permissions page for Manager Nusrat
Then reports.export is not offered to grant
And granting expenses.read.all to Nusrat succeeds and is audited
```

**US-09 Assistant guardrail**
```gherkin
Given I am Member Fahmida
When I ask "How much did the whole team spend this month?"
Then the assistant answers "You don't have access to team spending. You can ask about your own expenses."
When Admin Rahim asks "How much did Medicore spend?"
Then the assistant answers that it can only answer about TechCorp BD
```

**US-10 Language switch**
```gherkin
Given I am on page 2 of 10 of the Expenses list in English
When I switch to বাংলা
Then the title shows "খরচ", amounts show "৳১,২০০", and pagination shows "পৃষ্ঠা ২ / ১০"
And without reloading, every label, toast and validation message is in Bengali
```

**US-11 Editing can't bypass a Hard limit**
```gherkin
Given Travel has a Hard Limit budget of ৳30,000 and ৳20,000 is spent in June
And I saved a ৳1,200 Travel expense on 15 June (status "Saved")
When I edit its amount to ৳12,000 while June is open
Then the budget check runs with ৳20,000 − ৳1,200 + ৳12,000 = ৳30,800 against ৳30,000
And the expense becomes "Pending approval" and does not count in Spent
And the approvers receive "Approval Required"
When I edit it back to ৳1,200
Then it returns to "Saved" and the approvers receive "Request withdrawn"
```

**US-12 Team-scoped Manager and approvals**
```gherkin
Given Admin Rahim has narrowed Manager Nusrat to her team (Fahmida, Tarek)
When Karim, who is not in Nusrat's team, submits an expense over a Hard limit
Then Rahim receives "Approval Required" and Nusrat does not
And the expense does not appear in Nusrat's Approvals queue
And if Nusrat calls the approve API for it, she gets 404 Not Found
```

**US-13 Locked after approval or month close**
```gherkin
Given my June expense was approved by Nusrat
When I try to edit or delete it
Then I see "This expense is approved and can't be changed. Create an adjustment instead."
Given it is 5 July and June closed on 3 July
When my offline edit to a June expense syncs
Then it is refused with "June is closed" and I am offered "Create adjustment"
```

**US-20 / US-21 / US-22** from the requirement are covered by FR-RPT-06, FR-EXP-07 and FR-AST respectively.

---

## 8. Non-Functional Requirements (NFR)

| ID | Category | Requirement |
|---|---|---|
| NFR-PERF-01 | Performance | p95 API latency < 300 ms for cached reads and < 500 ms for writes at 500 RPS across 3 API replicas (reference: 2 vCPU/4 GB each) |
| NFR-PERF-02 | Performance | Monthly Summary p95 < 1 s with 1M expense rows in the tenant-month (pre-aggregation + cache) |
| NFR-PERF-03 | Performance | First load of the web app (PWA, 4G) < 2.5 s LCP; repeat visit < 1 s from the service-worker cache |
| NFR-SCAL-01 | Scalability | Stateless API: horizontal scale behind a load balancer; Redis backplane for SignalR and distributed cache |
| NFR-AVL-01 | Availability | 99.9% monthly for the API. Graceful degradation when Redis is down (falls back to in-process cache). When the DB is down, a friendly 503 with support contact. |
| NFR-SEC-01 | Security | OWASP ASVS L2 controls; OWASP Top 10 mitigations; TLS everywhere; secure headers; CORS allow-list |
| NFR-SEC-02 | Security | Rate limiting per user/tenant/IP, with stricter limits on auth and OTP endpoints |
| NFR-PRIV-01 | Privacy | Tenant isolation enforced in data access, cache keys, search, imports, exports, assistant and SignalR groups |
| NFR-OBS-01 | Observability | 5 categorised daily log files, correlation IDs, slow-query detection with caller location, health checks |
| NFR-MNT-01 | Maintainability | Clean Architecture + Vertical Slices + CQRS; SOLID, DRY, KISS, YAGNI; architecture tests enforce layer rules |
| NFR-UPG-01 | Upgradability | Backend upgrades from .NET 10 to .NET 11+ by changing one MSBuild property; frontend dependencies upgraded continuously by Renovate |
| NFR-PORT-01 | Portability | DB provider is switchable via configuration among InMemory, SQL Server, MySQL, Oracle, PostgreSQL, MS Access, MongoDB (tiered support, see TRD §6) |
| NFR-A11Y-01 | Accessibility | WCAG 2.2 AA; keyboard navigable; screen-reader labels in both languages |
| NFR-I18N-01 | Localisation | 100% string coverage checked in CI (missing-key test fails the build) |
| NFR-TEST-01 | Quality | ≥ 80% line coverage on Domain and Application; integration tests for every endpoint; load/stress tests per release |

---

## 9. Main Screens (summary — full specs in UiSpec.md)

From the requirement (8 main screens): Login · Organisation Onboarding · Dashboard · Add Expense · Manage Team · Budget Setup · Monthly Summary · Notifications.
Added by the extended requirements: MFA/OTP · Password reset/change · Workspace selector · Expenses list (advanced search) · Approvals · Adjustments · Categories · **Permissions** · Bulk Import wizard · Assistant panel · Profile & Security · Super Admin Platform Overview · System (Logs, Backups, Release Notes) · Offline/Error states.

---

## 10. Example Data (seed for Development & demo)

**TechCorp BD, June 2026 (Monthly Summary):**

| Category | Budget (৳) | Spent (৳) | Remaining (৳) | Status |
|---|---|---|---|---|
| Travel & Transport | 30,000 | 32,100 | −2,100 *(shown as "Over by ৳2,100")* | Over Budget |
| Meals & Entertainment | 25,000 | 20,400 | 4,600 | 82% |
| Software Subscriptions | 40,000 | 18,000 | 22,000 | On Track |
| Office Supplies | 20,000 | 9,400 | 10,600 | On Track |
| Training & Courses | 35,000 | 6,500 | 28,500 | On Track |
| **Total** | **1,50,000** | **86,400** | **63,600** | |

> Rule FR-ADJ-03 says "negative amounts never appear anywhere". The requirement's own mock shows "−৳2,100" in Remaining, so Outlay renders it as **"Over by ৳2,100"** in red (Q-03 resolved). The Total row keeps the arithmetic total, ৳63,600.

**Member-wise (June 2026):** Rahim Ahmed (Admin) ৳12,400 / 8 entries · Nusrat Jahan (Manager) ৳18,200 / 11 · Fahmida Akter ৳22,100 / 14 · Tarek Mahmud ৳19,800 / 12 · Others (4) ৳13,900 / 22 · **Total ৳86,400 / 67 entries**.

**Platform:** TechCorp BD (Pro, 8/15, Active, 01 Jun 2026) · Medicore Ltd (Starter, 3/3, Plan Limit Reached, 14 May 2026) · NovaSoft Inc (Enterprise, 47/∞, Active, 10 Apr 2026) · GreenLeaf Co (Starter, 2/3, Active, 20 Jun 2026).

---

## 11. Release Scope

| Release | Scope |
|---|---|
| **MVP (v0.5)** | All P0: sign-up, onboarding, roles, login + MFA, categories, expenses (own), yearly budgets with Soft Limit, 80% in-app alert, monthly summary, Super Admin org list, PWA shell, EN/BN |
| **v1.0 GA** | All P1: hard limit + approvals, adjustments, notifications via email/SMS, delegated permissions + UI, advanced search, bulk import, offline sync, assistant, ops APIs, Postman, full test suites, all 7 DB providers in their support tier |
| **v1.x** | P2: saved filters sharing, fuzzy duplicate detection, SSO, archive export |

See [ROADMAP.md](../ROADMAP.md).

---

## 12. Assumptions & Open Questions

Open questions are tracked in **MASTER-SPECIFICATION.md §6**, and limitations and deviations from the brief in **§7**, so they live in one place.

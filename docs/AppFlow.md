# Outlay — Application Flow (AppFlow)

| Field | Value |
|---|---|
| Document | AppFlow.md (2 of 6 core specs) |
| Version | 0.0.0 · 2026-10-28 |
| Purpose | Every user journey, step by step: screens, decisions, system checks, and what happens on failure. |
| Related | [PRD.md](PRD.md) (the *what*) · [UiSpec.md](UiSpec.md) (the *look*) · [TRD.md](TRD.md) (the *how*) |

> Flow IDs (`F-xx`) are referenced from UiSpec screens (`S-xx`) and Plan tasks. Diagrams use Mermaid (renders on GitHub/GitLab).

---

## 1. Navigation Map (route tree)

```mermaid
flowchart TD
    Start([Open app / PWA]) --> Auth{Signed in?}
    Auth -- No --> Login[/S-01 Login/]
    Login --> Signup[/S-05 Create Workspace wizard/]
    Login --> Forgot[/S-03 Forgot password/]
    Login --> Otp[/S-02 OTP verification/]
    Otp --> Multi{Member of >1 org?}
    Multi -- Yes --> Selector[/S-06 Workspace selector/]
    Multi -- No --> Home
    Selector --> Home
    Auth -- Yes --> Home{Role}
    Home -- Super Admin --> SA[/S-21 Platform overview/]
    Home -- Admin/Manager --> Dash[/S-07 Dashboard/]
    Home -- Member --> MyExp[/S-08 My expenses/]
    Dash --> Exp[/S-08 Expenses/]
    Dash --> Add[/S-09 Add expense/]
    Dash --> Sum[/S-14 Monthly summary/]
    Dash --> Team[/S-15 Team/]
    Dash --> Apr[/S-10 Approvals/]
    Team --> Perm[/S-16 Permissions/]
    Dash --> Bud[/S-13 Budget setup/]
    Dash --> Cat[/S-12 Categories/]
    Dash --> Notif[/S-17 Notifications/]
    SA --> Sys[/S-22 System: logs, backups, release notes/]
```

### 1.1 Route table

| Route | Screen | Default visibility (permission) |
|---|---|---|
| `/login` | S-01 Login | anonymous |
| `/login/verify` | S-02 OTP | partially authenticated (mfa_token) |
| `/forgot-password`, `/reset-password?token=` | S-03 | anonymous |
| `/signup` | S-05 Onboarding wizard | anonymous |
| `/invite/:token` | S-24 Accept invitation | anonymous |
| `/workspaces` | S-06 Workspace selector | authenticated |
| `/` → `/dashboard` | S-07 Dashboard | `dashboard.read` |
| `/expenses` | S-08 Expenses list | `expenses.read.own` |
| `/expenses/new`, `/expenses/:id` | S-09 Add/Edit | `expenses.create` / `expenses.update.own` |
| `/approvals`, `/approvals/:id` | S-10 | `expenses.approve` |
| `/adjustments` | S-11 | `adjustments.create` |
| `/categories` | S-12 | `categories.read` |
| `/budgets` | S-13 | `budgets.manage` |
| `/reports/monthly` | S-14 | `reports.monthly.read` |
| `/team` | S-15 | `users.read` |
| `/permissions` | S-16 | `permissions.manage` |
| `/notifications` | S-17 | `notifications.read` |
| `/imports/:resource` | S-18 Import wizard | `import.bulk` + resource create |
| (floating panel) | S-19 Assistant | `assistant.use` |
| `/me` | S-20 Profile & Security | authenticated |
| `/platform` | S-21 Platform overview | `tenants.read` (Super Admin) |
| `/system/logs`, `/system/backups`, `/system/release-notes` | S-22 | `system.logs.read`, `system.backup.download`, `system.releasenotes.read` |
| `*` | S-23 Not found / Offline / Service unavailable | — |

Menu items **only render if the user holds the permission**. Hidden is not the same as secure: the API re-checks every request.

---

## 2. F-01 Organisation Sign-Up & Onboarding

```mermaid
sequenceDiagram
    actor O as Owner
    participant W as Web (S-05)
    participant A as API
    participant DB as Database
    participant N as Email/SMS
    O->>W: Click "Create Workspace"
    W->>O: Step 1 Create Account (name, email, phone, password)
    O->>W: Submit
    W->>A: POST /api/v1/onboarding/account (Idempotency-Key)
    A->>A: Validate (email unique, password policy, phone E.164)
    A->>DB: Create User (BCrypt hash), status=PendingVerification
    A->>N: Send OTP (dev: 111111, nothing sent)
    W->>O: OTP screen with 05:00 countdown
    O->>W: Enter OTP
    W->>A: POST /api/v1/onboarding/verify
    W->>O: Step 2 Organisation (name, industry, team size, currency=BDT)
    W->>A: POST /api/v1/onboarding/organisation
    A->>DB: Create Tenant + Membership(role=TenantAdmin, all tenant permissions)
    W->>O: Step 3 Choose Plan (Starter / Pro / Enterprise=Contact Sales)
    W->>A: PUT /api/v1/onboarding/plan
    W->>O: Step 4 Invite Team (optional, plan seat counter)
    W->>A: POST /api/v1/invitations (0..n)
    A-->>W: Tokens issued for new tenant context
    W->>O: Dashboard + checklist: Create categories → Set budgets → Invite team
```

**Failure branches**
- Email already registered → "This email already has an Outlay account. Log in to create another workspace from the workspace menu." Existing users can create additional organisations after login.
- OTP expired → "Code expired" + Resend (enabled after 60 s).
- Enterprise chosen → creates the tenant on Starter limits, flags `UpgradeRequested=Enterprise`, and notifies Super Admin. Billing is out of scope.

---

## 3. F-02 Login, MFA, Lockout & Session

```mermaid
flowchart TD
    A[S-01 enter email + password] --> B[POST /connect/token grant_type=password]
    B --> C{Credentials valid?}
    C -- No --> D[Increment AccessFailedCount]
    D --> E{Count >= 3?}
    E -- Yes --> F[Lock account; revoke tokens; audit with IP; show Locked message + Reset link]
    E -- No --> G[Show 'Email or password is incorrect' + attempts left]
    C -- Yes --> H{Account locked?}
    H -- Yes --> F
    H -- No --> I{MFA required?}
    I -- No --> T[Issue tokens]
    I -- Yes --> J[Return error=mfa_required + mfa_token; send OTP via SMS + Email]
    J --> K[S-02 OTP screen with 5:00 countdown]
    K --> L[POST /connect/token grant_type=urn:outlay:otp]
    L --> M{OTP valid and not expired?}
    M -- No --> N{Attempts >= 5?}
    N -- Yes --> O[Invalidate code; force resend]
    N -- No --> K
    M -- Yes --> T
    T --> P{Memberships > 1?}
    P -- Yes --> Q[S-06 Workspace selector -> grant_type=urn:outlay:switch_tenant]
    P -- No --> R[Home for role]
```

**Rules**
- Failed counter resets on successful login. Lockout duration comes from `Auth:Lockout:DurationMinutes` (default 15).
- Message wording never reveals whether the email exists.
- **Session duration (API only):** `session_duration` (e.g., `PT4H`, `P1D`, or `4h`/`1d`) on the password grant sets the absolute session lifetime, clamped to `[Auth:Session:Min, Auth:Session:Max]`. The web UI never sends it, so the UI uses `Auth:Session:Default` (8 h).
- **Web token storage:** access token in memory only; refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie scoped to `/connect/token`. API clients such as Postman get the refresh token in the response body.
- **Silent refresh:** 60 s before access-token expiry the SPA calls the refresh grant. Rotation issues a new refresh token, and the old one becomes invalid. **Reuse of an old refresh token revokes the whole token family** (theft detection).

### 3.1 F-02b Password reset & change
1. Forgot password → enter email → always "If an account exists, we've sent instructions" → email link (valid 30 min) **and** OTP to phone.
2. Reset page → new password + confirm + OTP → password updated (BCrypt) → **all sessions revoked**, lockout cleared, audit entry.
3. Change password (S-20) → current password + new password + OTP → all *other* sessions revoked.

### 3.2 F-02c Logout
`POST /connect/revoke` (refresh token) + `/connect/logout` → clear in-memory token → **clear IndexedDB for this user/tenant** (configurable: "Keep offline data on this device" checkbox, default off on shared devices).

---

## 4. F-03 Add an Expense (Member)

```mermaid
flowchart TD
    A[S-09 open Add Expense] --> B[Pick date, category, amount, description]
    B --> C[Client validation: amount > 0, date <= today, category required]
    C --> D[Live Budget Impact panel: GET /budgets/impact?categoryId&amount&date]
    D --> E[Submit POST /api/v1/expenses with Idempotency-Key]
    E --> F{Server validation OK?}
    F -- No --> G[422 ProblemDetails: field errors shown inline]
    F -- Yes --> H{Exact duplicate?}
    H -- Yes, not confirmed --> I[409 duplicate_warning: show 'Looks like a duplicate of 14 Jun entry' + Save anyway]
    I -- Save anyway --> E2[Resubmit with confirmDuplicate=true]
    E2 --> J
    H -- No --> J{Budget for category/month?}
    J -- None --> K[Save: status=Saved]
    J -- Within budget --> K
    J -- Soft limit exceeded --> L[Save: status=Saved, IsOverBudget=true; notify Admins+Managers]
    J -- Hard limit exceeded --> M[Save: status=PendingApproval; notify the approvers immediately: Admins + Managers whose scope covers the member]
    K --> N{Crossed 80% this month first time?}
    L --> N
    N -- Yes --> O[Create 80% alert once per category per month]
    N -- No --> P[Done]
    O --> P
    M --> Q[Show 'Waiting for approval']
    P --> R[Show 'Expense saved' toast; update daily totals + invalidate caches]
```

**Offline variant:** if the network is down, the expense goes to the IndexedDB **outbox** with an Idempotency-Key and shows **Pending sync**. The budget check runs on the server at sync time, and the user is notified of the outcome (F-11).

### 4.1 F-03b Edit or delete an expense (owner)

```mermaid
flowchart TD
    A[S-09 open own expense] --> B{Saved or Pending, current year, month open?}
    B -- No: Approved/Rejected --> L1[Read-only + 'Create adjustment']
    B -- No: month closed --> L2[Read-only: 'June is closed' + 'Create adjustment']
    B -- Yes --> C[Edit date, category, amount, description]
    C --> D[Budget impact panel uses spend WITHOUT this expense's old amount]
    D --> E[PUT /api/v1/expenses/id with If-Match rowVersion + Idempotency-Key]
    E --> F{Version current?}
    F -- No --> G[409 conflict: show current state, e.g. 'Approved by Nusrat at 15:14']
    F -- Yes --> H{Validation OK and new date in an open month?}
    H -- No --> I[422 field errors or month_closed]
    H -- Yes --> J{Exact duplicate of ANOTHER expense?}
    J -- Yes, not confirmed --> K[409 duplicate_warning + Save anyway]
    K --> E
    J -- No --> M{Budget check for new category/month}
    M -- Within budget or unlimited --> N[Status Saved; if it was Pending: request withdrawn, approvers notified]
    M -- Soft limit exceeded --> O[Status Saved, Over budget flag on; if it was Pending: request withdrawn]
    M -- Hard limit exceeded --> P[Status Pending approval; approval request to approvers]
    N --> Q[Daily totals: old amount out, new amount in; alerts at most once per month; audit changed fields]
    O --> Q
    P --> Q
```

**Delete:** same eligibility as edit (own, Saved/Pending, open month) → `DELETE /api/v1/expenses/{id}` with `If-Match` → soft delete → amount leaves the totals → a pending approval request is cancelled and approvers notified → audited.

**Grace window:** between the 1st and the close date (default the 3rd), both months are open, so an edit can move an expense from June to July or back. Both months' totals update.

**Expense status lifecycle**

```mermaid
stateDiagram-v2
    [*] --> Saved: create within budget / Soft limit
    [*] --> PendingApproval: create over Hard limit
    Saved --> PendingApproval: owner edit exceeds Hard limit
    PendingApproval --> Saved: owner edit back within budget (request withdrawn)
    PendingApproval --> Approved: approver approves
    PendingApproval --> Rejected: approver rejects (reason)
    Saved --> Deleted: owner deletes (open month)
    PendingApproval --> Deleted: owner deletes (request cancelled)
    Approved --> [*]: locked, corrections via Adjustment
    Rejected --> [*]: locked, excluded from totals
    Deleted --> [*]: soft-deleted, excluded from totals
```


---

## 5. F-04 Over-Budget Approval (Manager)

1. Each **approver** receives **Approval Required** (real-time toast + notification + email/SMS on Pro+). Approvers are all Admins plus Managers with `expenses.approve` whose visibility covers the member (BR-NOT-04). A Manager narrowed to their team never receives requests for people outside it.
2. Opens S-17 Notifications or S-10 Approvals → **Review & Approve** → sees expense, member, category budget, month spent, pending total.
3a. **Approve** (optional note) → `POST /api/v1/expenses/{id}/approve` → status Approved, `ApprovedBy`, `ApprovedAt` → counted in Spent → member notified.
3b. **Reject** (reason **required**) → `POST /api/v1/expenses/{id}/reject` → status Rejected (excluded everywhere) → member notified with reason.
4. Concurrency: if two approvers act at once, the first wins. The second gets 409 "Already approved by Nusrat at 15:14".
5. A user cannot approve their own expense; it goes to another approver. If none exists, an Admin can self-approve, and that action is flagged in the audit log.
6. The approver group is evaluated when the queue loads and when approving. If an Admin narrows a Manager's scope while requests are pending, the out-of-scope requests disappear from that Manager's queue and approving them returns 404.
7. If the owner edits a Pending expense back within budget, or deletes it, the request is withdrawn or cancelled and approvers see it leave their queue (BR-EXP-09/10).

---

## 6. F-05 Invite a Team Member (Admin)

1. Admin opens S-15 Manage Team.
2. System checks seat usage. If the limit is reached, the invite form is replaced by an upgrade prompt and the invitation is blocked (BR-PLAN-01).
3. Admin enters email, role and (optionally) **permissions to grant** (subset of the Admin's own).
4. `POST /api/v1/invitations` → invitation email with a link that expires in **48 h**.
5. Invitee opens `/invite/:token` → sets name, phone and password (or, if they already have an Outlay account, logs in and accepts) → OTP verify → joins.
6. Admin sees status change **Invited → Active** (real-time) and a "New Member Joined" notification is created.
7. Expired → status "Expired" → **Resend** creates a new token and invalidates the old one.

---

## 7. F-06 Budget Setup (Admin)

1. S-13 lists **all active categories** for the selected year (default: current year).
2. For each row: amount (৳, decimals allowed) + limit type (Soft/Hard; Hard is disabled on Starter with a "Pro feature" tooltip). Empty means **Unlimited**.
3. Save → `PUT /api/v1/budgets/{year}` (batch upsert) → validation → audit → cache invalidation → budget monitor recalculates current month status (may raise 80%/over alerts *once*).
4. Previous years are shown read-only with a lock icon.

---

## 8. F-07 Adjustment / Correction

1. Member or Admin opens an expense → **Create adjustment** (or S-11).
2. Links the original expense. Enters direction (Reduction e.g. refund / Increase), amount (> 0), reason, date.
3. Validation: a Reduction cannot exceed the original amount net of earlier reductions. Date ≥ original date and not in the future.
4. Saved as a separate record. The original is untouched. Reports net it into the adjustment's month.

---

## 9. F-08 Monthly Summary & Export

1. Manager/Admin opens S-14 → filters: month (default current), member (Pro+), category.
2. `GET /api/v1/reports/monthly?year=2026&month=6&memberId=` → served from cache → else from **DailyCategoryTotals** (pre-aggregated) → else live aggregate. Closed months come from **MonthlySnapshots**.
3. Toggle **View by Member** → member-wise table.
4. **Export Report** → `GET /api/v1/reports/monthly/export?format=xlsx|csv` (Pro+). Starter sees an upgrade tooltip.

---

## 10. F-09 Advanced Search (all CRUD lists)

```mermaid
flowchart LR
    A[Quick search box] -->|free text| Q[Build query]
    B[Filter builder: field + operator + value rows, AND/OR groups] --> Q
    C[OQL text: category = Travel AND amount >= 1000] -->|parse + validate| Q
    Q --> R[POST /api/v1/expenses/search]
    R --> S{Valid fields/operators?}
    S -- No --> T[400 with pointer to bad clause, highlighted in UI]
    S -- Yes --> U[Paged results + total + localised pagination]
    U --> V[Save filter as... optional]
```

- The filter state is serialised into the URL (`?q=` base64url), so filtered views can be shared by link.
- Field and operator lists come from `GET /api/v1/{resource}/search/fields`, so the UI builder is generic.

---

## 11. F-10 Bulk Import (CSV / Excel)

```mermaid
sequenceDiagram
    actor U as User
    participant W as Web (S-18)
    participant A as API
    participant Q as Background worker
    participant DB as Database
    U->>W: Choose resource, download template (EN/BN headers)
    U->>W: Upload .csv / .xlsx
    W->>A: POST /api/v1/expenses/import (multipart, Idempotency-Key)
    A->>A: Check size/type (magic bytes), create ImportJob
    alt rows <= SyncThreshold (default 1000)
        A->>A: Parse + validate inline
    else large file
        A-->>W: 202 Accepted {jobId}
        A->>Q: Enqueue job
        Q-->>W: SignalR progress events (parsed/validated/total)
    end
    Note over A,Q: Validate ALL rows in parallel: types, FluentValidation (same validators as single create),<br/>in-file duplicates, DB duplicates (batched), business rules, plan limits
    alt any error
        A-->>W: 422 import_failed + errors[{row, column, value, code, message}]
        W->>U: Error grid (row/column/reason) + Download annotated file
        Note over DB: Nothing written (validate-all-then-commit)
    else all valid
        A->>DB: Single transaction, chunked inserts
        A-->>W: 200 {inserted: n}
    end
```

The user fixes the file and re-uploads. A new Idempotency-Key is used because the content has changed.

---

## 12. F-11 Offline Mode & Sync

```mermaid
stateDiagram-v2
    [*] --> Online
    Online --> Offline: navigator offline / network error / 5xx streak
    Offline --> Online: online event + GET /health/ready OK
    state Offline {
        [*] --> ReadCache
        ReadCache: Reads served from IndexedDB
        ReadCache --> QueueWrite: user creates/edits/deletes
        QueueWrite: Outbox item (Idempotency-Key, base rowVersion)
        QueueWrite --> ReadCache
    }
    Online --> Syncing: outbox not empty
    Syncing --> Online: all items 2xx
    Syncing --> Conflict: 409 rowVersion mismatch
    Syncing --> Rejected: 422 validation / 403 permission
    Conflict --> Online: user picks Keep mine (re-submit) or Use server
    Rejected --> Online: user edits or discards
```

1. Replay is FIFO per entity. Each outbox item carries its original Idempotency-Key, so a retry after a timeout never duplicates.
   - An edit or delete for a month that closed while the device was offline comes back as `month_closed`. The sync panel shows "June is closed" with a **Create adjustment** button pre-filled from the rejected change (BR-EXP-12).
2. After the push, a **delta pull**: `GET /api/v1/sync/changes?since=<cursor>` returns changed records and tombstones within the user's scope.
3. A badge shows the pending count. The sync history panel lists the result of each item.

---

## 13. F-12 Permission Delegation

```mermaid
flowchart TD
    A[S-16 Permissions: select person in hierarchy tree] --> B[GET /api/v1/permissions/users/:id effective + grantable]
    B --> C[Matrix: toggles enabled only for permissions the actor holds]
    C --> D{Grant or revoke?}
    D -- Grant --> E[POST /api/v1/permissions/users/:id/grants]
    E --> F{Actor holds every permission being granted?}
    F -- No --> G[403 cannot_grant_unheld_permission]
    F -- Yes --> H[Save grants GrantedBy=actor; bump PermissionVersion; invalidate cache; audit]
    D -- Revoke --> I[GET .../revoke-preview -> list of downstream users affected]
    I --> J[Confirm dialog listing affected people]
    J --> K[DELETE grants + cascade down grant chain; revoke affected users' refresh tokens; audit]
```

The Super Admin uses the same page from S-21 to manage Admins' permissions, but never sees tenant expense data.

---

## 14. F-13 Assistant (Text & Voice)

```mermaid
sequenceDiagram
    actor U as User
    participant W as Web (S-19)
    participant S as Browser Speech API
    participant A as API /assistant/ask
    participant P as Intent engine (deterministic)
    participant G as Guardrail + Authorization
    participant Q as Existing MediatR queries
    U->>W: Tap mic (or type)
    W->>S: SpeechRecognition (lang = en-US or bn-BD)
    S-->>W: Transcript
    W->>A: POST {text, locale}
    A->>P: Normalise (Bengali digits, month names), match intent + slots
    P-->>A: Intent=TopSpenderThisMonth, slots{period=2026-06}
    A->>G: Check permission + data scope (tenant from token only)
    alt not allowed / foreign tenant / super admin data
        G-->>A: Refusal template
    else allowed
        A->>Q: Send GetMemberSpendingQuery (same as Monthly Summary)
        Q-->>A: Data
    end
    A-->>W: {answerText, answerSsml?, data, intent, suggestions}
    W->>S: speechSynthesis.speak(answer, lang)
    W->>U: Text answer + optional mini table/chart
```

If no intent matches, the assistant returns "I can help with…" plus 4 localised example questions. If speech isn't supported (e.g., Firefox), the mic button is hidden and text chat still works.

---

## 15. F-14 Language Switch

1. User selects **EN / বাং** in the top bar (or on the login screen).
2. Frontend: `i18next.changeLanguage('bn')` → `document.documentElement.lang='bn'` → `document.title` re-rendered → all `Intl` formatters rebuilt → TanStack Query refetches the queries that carry server-localised text.
3. Backend: `PATCH /api/v1/me/preferences {language:'bn'}` (saved). Subsequent requests send `Accept-Language: bn-BD`, so validation and error messages come back in Bengali.

---

## 16. F-15 Dependency Failure & Graceful Errors

| Situation | Backend behaviour | Frontend behaviour |
|---|---|---|
| Redis down | Circuit opens → cache falls back to in-process memory → requests succeed → warning logged once per minute | Nothing visible |
| DB down / timeout after retries | 503 ProblemDetails `code=DEPENDENCY_UNAVAILABLE` + support contact | Full-page or inline "Something went wrong. Please contact **{Support.Name}** at **{Phone}** / **{Email}**" (localised) + reference ID (correlation ID) + switch to offline mode |
| Email/SMS provider down | Message stays in outbox, retried with backoff; OTP screen says "We couldn't send the SMS, try email or resend" | Countdown and alternative channel |
| Unhandled exception | 500 ProblemDetails (no stack trace), exception log with class/file/method/line + probable cause + fix | Friendly error + reference ID |
| Rate limited | 429 + `Retry-After` | "Too many attempts. Try again in N seconds." with countdown |
| Backend unreachable at first load | — | Support contact from build-time env fallback (`VITE_SUPPORT_*`), cached IndexedDB data if any |

---

## 17. F-16 PWA Install

1. The service worker registers on first load. The manifest is linked.
2. On Chromium, `beforeinstallprompt` is captured, and after the second visit or the first successful expense a **"Install Outlay as an app"** banner appears (dismissible, snoozed 14 days).
3. iOS Safari has no install prompt API, so a one-time hint is shown: "Tap Share → Add to Home Screen".
4. New version available → toast "A new version of Outlay is ready — Refresh" (`skipWaiting` on confirm).

---

## 18. F-17 Super Admin Operations

- **Platform overview** (S-21): organisations, plan, seat usage, status, joined date; actions: change plan, suspend/reactivate, create organisation + Admin, manage Admin permissions.
- **Logs** (S-22): pick category + date (dd-MM-yyyy) → preview the tail (slow queries highlighted red) → download.
- **Backups** (S-22): "Create backup" → job → download when ready. Each action requires a fresh OTP (step-up) and is audited.
- **Release notes** (S-22, also for SQA): latest and previous versions with "What to test" checklists.

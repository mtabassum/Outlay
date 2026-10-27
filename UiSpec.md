# UiSpec — Outlay Frontend

Stack: **Next.js (latest stable) · React 19+ · TypeScript · Tailwind CSS v4**. Offline-first PWA. English + Bengali.

## 1. Design language — "Emerald Ledger"

| Token | Light | Dark | Use |
|---|---|---|---|
| `--brand-600` | #059669 | #10b981 | Primary actions, brand |
| `--brand-900` | #064e3b | #022c22 | Hero, sidebar gradient |
| `--accent` (amber) | #f59e0b | #fbbf24 | Warnings, 80% alerts |
| `--danger` | #dc2626 | #f87171 | Over budget, destructive |
| `--info` | #2563eb | #60a5fa | Info, links |
| `--violet` | #7c3aed | #a78bfa | Super Admin / Enterprise |
| `--surface` | #ffffff | #0f1512 | Cards |
| `--bg` | #f4f7f5 | #0a0f0d | Page |

* **Look**: soft glass cards (`backdrop-blur`, 1 px translucent borders), emerald→teal gradient hero/sidebar, generous radius (`rounded-2xl`), layered shadows, animated progress rings, subtle motion (150–250 ms, respects `prefers-reduced-motion`).
* **Signature element**: the **Budget Ring** (radial gauge) — green < 80%, amber 80–99%, red ≥ 100%; the same colour language is used in bars, chips and rows everywhere.
* **Typography**: Inter (Latin) + Noto Sans Bengali; tabular numerals for money; scale 12/14/16/20/24/32.
* **Icons**: lucide-react. **Charts**: lightweight (Recharts or custom SVG).
* **Dark mode**: system default + toggle; all tokens defined as CSS variables in `@theme`.
* **Accessibility**: WCAG 2.2 AA contrast, visible focus rings, keyboard navigable tables/dialogs, `aria-live` for toasts and OTP countdown, 44 px touch targets.
* **Responsive**: mobile-first; breakpoints 360 / 640 / 1024 / 1280. Sidebar → bottom tab bar on mobile. Tables → cards under 640 px.

## 2. App shell
* **Top bar**: workspace switcher (shows org + role chip), global search (⌘K opens command palette incl. assistant), **language switch (EN | বাংলা)**, theme toggle, online/offline/sync indicator, notifications bell with unread count, profile menu.
* **Sidebar** (permission-filtered — items the user can't access are not rendered): Dashboard · Expenses · Add Expense · Categories · Budgets · Monthly Summary · Team · Permissions · Notifications · Assistant · (Super Admin: Organisations · Plans · Operations · Release Notes).
* **Banners**: offline ("You're offline — changes will sync"), sync progress, dependency-down notice with support contact, install-app prompt.

## 3. Shared components (build once, reuse everywhere — DRY)
| Component | Notes |
|---|---|
| `DataTable` | server paging, sorting, column picker, sticky header, row actions, empty/error/loading states, localized pagination ("১–২০ / ৬৭") |
| `AdvancedFilterBar` | field ▸ operator ▸ value rows, AND/OR groups, chips, save/load filter, URL-synced; operators per field type |
| `BulkImportDialog` | template download, drag-drop CSV/XLSX, progress, **error grid (row/column/message) + download error report**, re-upload |
| `FormField` | label, hint, error (localized), required marker; RHF + zod |
| `MoneyText` | formats with org currency & active locale (৳ ১,২০০.০০ / ৳1,200.00) |
| `BudgetRing`, `BudgetBar`, `StatusChip` | colour language |
| `ConfirmDialog`, `Toast`, `Skeleton`, `EmptyState`, `OtpInput` (6 boxes + countdown), `PermissionTree`, `SupportContactCard` |

## 4. Screens

### 4.1 Login (entry)
Two-pane: left emerald gradient with product tagline + rotating tips; right card: Email, Password (show/hide), **Log in**, Forgot password. Below: "New organisation? Create workspace". Dev banner (only when `NEXT_PUBLIC_ENV=development`): sample accounts and "OTP is 111111".
States: invalid credentials (generic), locked, MFA step (OTP boxes, **mm:ss countdown**, Resend after 60 s), workspace selector (cards with org, role chip, last used).

### 4.2 Onboarding wizard (4 steps)
Stepper: ✓ Create Account → Set Up Organisation (name*, industry, team size, currency) → Choose Plan (3 cards, Pro highlighted "Most popular", comparison table) → Invite Team (email+role rows, seat meter) → "Create your first categories & budgets" checklist.

### 4.3 Dashboard (Admin)
KPI tiles: Team Spent · Total Budget · Team Members `8/15` · Pending Approvals (amber, clickable). **Budget Health** list of bars with `spent / budget — %` and colour state. **Recent expenses** (Date, Member, Category, Description, Amount, Status chip). Quick actions: Add Expense · Monthly Summary · Manage Team. Manager sees same minus billing/team-manage; Member sees personal version (own spend, remaining per category, own recent).

### 4.4 Add Expense
Date (≤ today), Category (active only), Amount (>0, decimals), Description. Live **Budget Impact** panel: 80% warning (amber), over-budget Soft (amber, "will be marked Over Budget"), Hard (red, "needs Manager/Admin approval"; button label becomes **Submit for Approval**). Duplicate warning with "Save anyway". Offline: button "Save offline". Info strip: "As a Member you can only view and edit your own expenses."

### 4.5 Expenses list (all roles, scope differs)
DataTable + AdvancedFilterBar + **Import** + **Export** + Add. Row actions: view, edit (own, current year), delete (own, current year), **Add adjustment**. Detail drawer shows timeline: created → alerts → approval → adjustments.

### 4.6 Categories
Table (Name, Status, Used this month, Budget). Create/Edit modal (name ≤ 50, case-insensitive uniqueness error inline). Deactivate/Reactivate. Starter at 10 ⇒ disabled Add with upgrade CTA. Import/export.

### 4.7 Budget Setup (Admin)
Year selector (past years read-only, lock icon). Grid: Category · Monthly budget (decimal) · Limit type (Soft/Hard; Hard disabled with lock + tooltip on Starter) · Status (Set / Unlimited). **Save Budget**, **Reset Changes**. "Categories without a budget are Unlimited — no alerts."

### 4.8 Manage Team (Admin)
Invite row (Email, Role, **Send Invite**) + seat meter ("Plan: Pro — 8 of 15 seats used. 7 left"). Team table (Name, Email, Role, Status Active/Invited/Locked, Actions: Change role, Remove, Resend, Unlock). Search + filters + bulk import users.

### 4.9 Permissions (dedicated page)
Left: member list (grouped by role). Right: **PermissionTree** (resource ▸ actions) with tri-state checkboxes. Only permissions the viewer holds are enabled; others are greyed with "You don't have this permission". Shows "Granted by", "Effective since", unsaved-changes bar, cascade warning dialog ("Removing this also removes it from 3 people they delegated to"). Audit link.

### 4.10 Monthly Summary (Manager & Admin)
Filters: Member (All/one), Month. KPI tiles: Total Spent · Total Budget · Remaining · Over Budget count. Category table with status chips (Over Budget red row, "82% Used" amber, On Track green). Tabs: **By Category | By Member**. Buttons: Export (CSV/XLSX/PDF), View by Member. Last-computed timestamp when served from snapshot/cache.

### 4.11 Notifications (all roles)
List with severity colour bars: Approval Required (red, **Review & Approve**), Over Budget (red), 80% Alert (amber), New Member (blue), Seats (amber). **Mark all as read**. Approval modal: details, budget status, note/reason, Approve | Reject.

### 4.12 Assistant
Chat panel (drawer or page). Text input + **mic button** (hold/click to speak) + speaker toggle for spoken replies. Quick-prompt chips ("How much did we spend this month?"). Answers may include a small table. Shows scope badge ("Answering for: TechCorp BD · Manager scope"). Clear error text when mic unsupported.

### 4.13 Super Admin
* **Organisations**: table (Org, Plan chip, Users `8/15`, Status [Active / Plan Limit Reached], Joined) + advanced filters; create org / provision Admin; no expense data anywhere.
* **Plans**: edit limits/prices (data-driven).
* **Operations**: Download DB backup (shows active provider), Download logs (category dropdown + date picker `dd-mm-yyyy`), health panel (DB, Redis, SMS, SMTP), recent audit.
* **Release Notes**: latest + history with "What QA should verify" checklist.

### 4.14 Settings / Profile
Change password, MFA status/phone/email verification, language preference, sessions list with revoke. (No "session time" field — API only.)

### 4.15 Error & edge pages
`/offline` (precached), 403 ("You don't have access — ask your Admin"), 404, dependency-down page with **SupportContactCard** (name, phone, email from env), maintenance.

## 5. Localization rules (full-screen switch)
* Every string via message keys (`en.json`, `bn.json`); **no hard-coded text**. Page `<title>`, `aria-labels`, placeholders, validation messages, toasts, empty states, table headers, tooltips, email subjects.
* Numbers, dates, currency, percentages and **pagination text** via `Intl.*` with `bn-BD` (Bengali digits ০–৯) / `en-US`. Locale in URL (`/en/...`, `/bn/...`) + cookie; switching preserves route and filters.
* Server messages arrive as **error codes + localized text** (`Accept-Language`).
* Layout tested with Bengali's taller line-height (+10%).

## 6. PWA & offline UX
* `public/manifest.json`: name Outlay, short_name Outlay, `display: standalone`, theme `#059669`, icons 192/512 + maskable, shortcuts (Add Expense, Dashboard).
* `public/service-worker.js`: precache shell + `/offline`; navigation network-first with offline fallback; static assets cache-first (versioned); API GETs stale-while-revalidate for allow-listed endpoints; **never caches** auth endpoints or responses with `Cache-Control: no-store`.
* **Install prompt**: capture `beforeinstallprompt`, show a dismissible "Install Outlay" card (and iOS Safari "Share → Add to Home Screen" hint); also a native-style notification-like banner on first successful visit.
* Update flow: new SW waiting ⇒ toast "New version available — Refresh".

## 7. Page titles (all localized)
`Login · Create workspace · Dashboard · Expenses · Add expense · Categories · Budgets · Monthly summary · Team · Permissions · Notifications · Assistant · Organisations · Operations · Release notes · Settings` formatted `"{page} — Outlay"` / `"{পৃষ্ঠা} — আউটলে"`.

## 8. Acceptance checklist for any screen
- [ ] Both languages complete incl. title & pagination  - [ ] Dark + light  - [ ] 360 px layout  - [ ] Keyboard only  - [ ] Loading/empty/error/offline/forbidden states  - [ ] Role-gated  - [ ] No money in console/logs  - [ ] Uses shared components

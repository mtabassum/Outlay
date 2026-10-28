# Outlay — UI Specification (UiSpec)

| Field | Value |
|---|---|
| Document | UiSpec.md (3 of 6 core specs) |
| Version | 0.0.0 · 2026-10-28 |
| Stack | React 19 + TypeScript + Vite + **Tailwind CSS v4** (tokens via `@theme`) + Radix UI primitives + lucide-react icons + Recharts |
| Related | [PRD.md](PRD.md) · [AppFlow.md](AppFlow.md) (flow IDs `F-xx`) · [TRD.md](TRD.md) §12 (frontend architecture) |

---

## 1. Design Direction — "Kantha Ledger"

**Subject:** a bilingual (English/বাংলা) expense and budget tool for small and mid-size organisations in Bangladesh and beyond. Its users are busy admins, managers and employees who record money on phones and laptops.

**Primary job of the UI:** show budget health at a glance and make adding an expense take under 15 seconds.

**Concept:** *Nakshi kantha* is the Bengali quilt stitched from many small running stitches, and that is the metaphor for a ledger built from many small expenses. The interface is calm, cool and precise, with **one** memorable element: the **Budget Thread**. Budget progress is drawn as a stitched thread, not a generic bar. Everything else stays quiet.

### 1.1 What makes it unique (and what we avoid)

| We do | We deliberately avoid |
|---|---|
| A cool indigo "ink and paper" palette with paddy-green, turmeric and shapla-rose status colours | Cream backgrounds with terracotta accents; near-black with a single acid accent; the dark-green clone of the requirement mock |
| One typeface family designed for **both** Latin and Bengali (Anek), so EN↔BN switching keeps the same personality | Mixing an unrelated Bengali fallback font that changes the look on switch |
| Budget Thread as the single signature element | Identical rounded cards with the same grey shadow everywhere; gradient washes |
| Sentence-case labels, plain verbs | ALL-CAPS eyebrow labels, "→" appended to buttons, middle-dot meta strings |
| Hierarchy with spacing and type weight, with borders only where they carry meaning | Decorative dividers and numbered markers on non-sequential content |

---

## 2. Design Tokens

Tokens live in `src/Frontend/outlay-web/src/styles/theme.css` as Tailwind v4 `@theme` variables, so they are available as utilities (`bg-ink`, `text-paddy`, …) **and** as CSS variables. No hard-coded hex values are allowed in components (lint rule).

### 2.1 Colour

| Token | Light | Dark | Use |
|---|---|---|---|
| `--color-ink` | `#1E2A5A` | `#C9D2F5` | Primary text on paper, primary buttons, nav active state |
| `--color-ink-soft` | `#4A5683` | `#9AA5D1` | Secondary text, icons |
| `--color-paper` | `#EEF1F8` | `#121733` | App background (cool mist; dark mode is indigo-tinted, never neutral black) |
| `--color-surface` | `#FFFFFF` | `#1A2045` | Panels, tables, sheets |
| `--color-line` | `#D5DBEA` | `#2C3463` | Borders that separate data (table rows, inputs) |
| `--color-paddy` | `#2F8F5B` | `#5CC48A` | On track, success, "Saved" |
| `--color-turmeric` | `#E0A019` | `#F2BE4A` | 80% warning, pending sync |
| `--color-shapla` | `#C92A4B` | `#F06C86` | Over budget, errors, destructive actions |
| `--color-jamdani` | `#5B4FD6` | `#9B92FF` | Links, focus ring, info, assistant accent |

Contrast: every text/background pair is ≥ 4.5:1 (checked by a Storybook a11y addon in CI). Status is **never** shown by colour alone; there's always a label or icon ("Over budget", ⚠).

### 2.2 Typography

| Role | Font | Size / line-height | Weight |
|---|---|---|---|
| Display (page titles, KPI numbers) | **Anek Latin** / **Anek Bangla** (variable width + weight, Google Fonts) | 32/36 (mobile 26/30) | 650, width 110 (slightly expanded) |
| Heading 2 | Anek | 22/28 | 600 |
| Heading 3 | Anek | 18/24 | 600 |
| Body | Anek | 16/24 (Bengali 16/26, more leading for conjuncts) | 400 |
| Small / table | Anek | 14/20 | 450 |
| Numbers | Anek with `font-variant-numeric: tabular-nums` (fallback: `Noto Sans Bengali` / `Noto Sans` tabular) | — | 500 |

- Scale ratio is 1.2 (minor third), following *The Elements of Typographic Style*. Body line length ≤ 75 characters (`max-w-[68ch]` on prose).
- The `lang` attribute switches the `font-family` stack: `:lang(bn) { font-family: "Anek Bangla", "Noto Sans Bengali", sans-serif; }`.
- Fonts are self-hosted via `@fontsource-variable/anek-latin` and `@fontsource-variable/anek-bangla` so they work offline in the PWA.

### 2.3 Space, radius, elevation, motion

| Token | Value |
|---|---|
| Spacing scale | 4-pt: 4, 8, 12, 16, 24, 32, 48, 64 |
| Radius | `--radius-input: 8px`, `--radius-panel: 14px`, `--radius-pill: 999px` (status chips only). Hierarchy decides the radius; it isn't one radius everywhere. |
| Elevation | Panels sit flat on paper with a 1px `--color-line` border. Only **floating** layers (menus, dialogs, assistant, toasts) get a shadow: `0 12px 32px -12px rgb(30 42 90 / .35)` |
| Motion | 150 ms ease-out for state changes, 240 ms for sheets/dialogs. **One** orchestrated moment: the Budget Thread "stitches in" on first dashboard load. `prefers-reduced-motion` disables it. |

### 2.4 The Budget Thread (signature component)

```
Travel & Transport                                   ৳32,100 / ৳30,000  Over by ৳2,100
╺━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━●━━━━━━━━━┓░░░  (shapla, overflow knot)
Meals & Entertainment                                ৳20,400 / ৳25,000  82%
╺━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ●┄┄┄┄┄┄┄┄┄  (turmeric past the 80% knot)
Software Subscriptions                               ৳18,000 / ৳40,000  45%
╺━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄●┄┄┄┄┄┄┄┄┄  (paddy)
                                                      ↑ 80% knot
```

- Implemented as SVG. The spent portion is a `stroke-dasharray` "running stitch" (6 on / 3 off), the remaining track is a faint dotted guide, and a small **knot** marks the 80% threshold.
- Overspend continues past the end as a frayed shapla segment with a `+৳2,100` label. It never shows a negative number (Q-03, resolved).
- Accessible: `role="meter"` with `aria-valuenow`, `aria-valuemax`, and an `aria-valuetext` such as "Travel and Transport, 107 percent, over budget by 2,100 taka" (localised).

---

## 3. Layout & Responsiveness

### 3.1 App shell (≥ 1024 px)

```
┌──────────────┬───────────────────────────────────────────────────────────────┐
│ Outlay       │ [TechCorp BD ▾]   Search (/)          ● Synced  EN|বাং  🔔3  (R)│
│              ├───────────────────────────────────────────────────────────────┤
│ Dashboard    │  Page title                                     [Primary CTA] │
│ Expenses     │                                                               │
│ Approvals 2  │  Content (12-col grid, max-width 1280, left-aligned)          │
│ Reports      │                                                               │
│ Budgets      │                                                               │
│ Categories   │                                                               │
│ Team         │                                                               │
│ Permissions  │                                                    ┌────────┐ │
│ ───────      │                                                    │Ask     │ │
│ Settings     │                                                    │Outlay ◉│ │
└──────────────┴────────────────────────────────────────────────────┴────────┘─┘
```

- Left rail is 248 px and collapses to a 72 px icon rail. Items render only if the user holds the permission.
- Top bar: workspace switcher, global search (`/` shortcut), **sync status** (Synced / Offline / n pending), language toggle `EN | বাং`, notifications bell with count, avatar menu.
- Content is **left-aligned**, with numbers right-aligned in tables.
- The **Ask Outlay** assistant is a floating button bottom-right that opens a 400 px side sheet.

### 3.2 Breakpoints

| Name | Width | Behaviour |
|---|---|---|
| `sm` | < 640 | Bottom tab bar: Home · Expenses · **＋ Add** (centre, raised) · Reports · More. Tables become stacked list rows. Filters open in a bottom sheet. |
| `md` | 640–1023 | Icon rail, 2-column KPI grid |
| `lg` | 1024–1439 | Full rail, 4-column KPI grid |
| `xl` | ≥ 1440 | Content max-width 1280 px, centred in the paper area |

Touch targets ≥ 44×44 px. There's no horizontal page scroll at any width (tables scroll inside their own container with a sticky first column).

---

## 4. Component Library (`src/Frontend/outlay-web/src/shared/ui`)

| Component | Notes |
|---|---|
| `Button` | Variants: primary (ink), secondary (outline), quiet (text), danger (shapla). Loading state keeps the width. Labels are verbs: "Save expense", "Send invite". |
| `Input`, `MoneyInput`, `DateInput`, `Select`, `Combobox` | `MoneyInput` shows ৳, accepts both Bengali and Latin digits, stores a decimal string, and formats with South-Asian grouping on blur |
| `StatusChip` | Pill with icon + text: Saved, Over budget, Pending approval, Approved, Rejected, Invited, Active, Expired, Pending sync |
| `BudgetThread` | Signature component (§2.4) |
| `KpiStat` | Label above, big tabular number, optional delta. Used sparingly (dashboard and summary only). |
| `DataTable` | TanStack Table. Server-side sort and pagination, column visibility, row selection for bulk actions, sticky header, localised pagination ("Showing 21–40 of 67" / "৬৭টির মধ্যে ২১–৪০ দেখানো হচ্ছে") |
| `FilterBuilder` | Advanced search (§6.3) |
| `ImportWizard` | Bulk import (§6.9) |
| `OtpInput` | 6 boxes, paste support, auto-submit, Bengali digits accepted, **countdown ring** 05:00 → 00:00, Resend enabled after 60 s |
| `ProblemAlert` | Renders RFC 9457 ProblemDetails: localised title, field errors, **reference ID** (correlation ID), support contact for 5xx |
| `EmptyState` | One sentence + the action that fills it ("No expenses this month. Add your first expense.") |
| `OfflineBanner` | Turmeric bar: "You're offline. Changes will sync when you're back. 3 pending." |
| `InstallPrompt` | PWA install banner (§9) |
| `ConfirmDialog` | Destructive confirmations name the object: "Remove Karim Hasan from TechCorp BD?" |

---

## 5. Copy & Voice Rules

- Sentence case everywhere. No ALL-CAPS labels.
- Buttons say what happens: "Submit for approval", "Approve expense", "Reject expense", "Send invite", "Save budget". The resulting toast reuses the verb: "Expense submitted for approval".
- Errors explain what happened and how to fix it, and don't apologise: "Amount must be more than ৳0." / "This category already exists. Choose another name."
- Service failures (5xx): "Something went wrong on our side. Contact the Outlay team at +880 1XXX-XXXXXX or support@outlay.app and share reference **8f3c-21a0**." The name, phone and email come from config (`GET /api/v1/system/support-contact`).
- Every string is an i18n key. English is the source and Bengali the translation. Strings are never concatenated; use ICU messages with plurals (`{count, plural, one {# member} other {# members}}`).

---

## 6. Screen Specifications

Each screen lists: purpose, who sees it, layout, components, states (loading / empty / error / offline), and the flow it serves.

### S-01 Login — F-02
- **Layout:** split view on ≥ lg. Left: the form panel on surface. Right: a large Budget Thread illustration stitching the word "Outlay", static under reduced motion. On mobile, the form only.
- **Fields:** Email, Password (show/hide). **Log in**, **Forgot password?**. Below that: "New organisation? **Create workspace**" (opens S-05).
- **States:** wrong credentials → inline "Email or password is incorrect. 2 attempts left." After 3 attempts the **Locked** panel shows the reset link and the time remaining.
- **Dev ribbon:** in Development a jamdani ribbon "Development — OTP is 111111" plus quick-fill buttons for the seeded demo accounts. Not rendered in other environments.
- There's **no session-length option** (API-only feature).

### S-02 OTP verification — F-02
- Heading "Enter the 6-digit code". Sub-text: "Sent to •••• 4821 and r•••@techcorp.com.bd".
- `OtpInput` + **countdown ring** (05:00). At 00:00 the inputs disable and "Code expired" shows with a **Send a new code** button.
- **Resend** is disabled for the first 60 s and shows "Resend in 0:42".
- Errors: "That code isn't right. 3 tries left."

### S-03 Forgot / Reset password, S-04 Change password — F-02b
- Reset: new password + confirm + strength meter (zxcvbn-ts) + OTP. Policy hints are shown live (12+ chars, not a breached password).
- Change (inside S-20): current password + new + OTP. On success: "Password changed. Other devices have been signed out."

### S-05 Create Workspace wizard — F-01
- Stepper with 4 steps: **Create account → Set up organisation → Choose plan → Invite team**. The steps are a real sequence, so they're numbered.
- Step 2: Organisation name*, Industry (select), Team size (1–5, 6–15, 16–50, 50+), Currency (default BDT ৳), Time zone (default Asia/Dhaka).
- Step 3: three plan columns (Starter / Pro *Most popular* / Enterprise) with the comparison rows Price, Users, Categories, History. Buttons: **Select**, **Select Pro**, **Contact sales**.
- Step 4: email + role rows, with the seat counter "Pro: 1 of 15 seats used". **Skip for now** is allowed.

### S-06 Workspace selector
- A list of the user's organisations with role chip, plan, and last-used date. Selecting one exchanges the token and goes to that org's home.
- Also reachable any time from the top-bar switcher.

### S-07 Dashboard (Admin/Manager) — F-08
- Heading: "TechCorp BD, June 2026 overview".
- **KPI row (4):** Team spent ৳86,400 · Total budget ৳1,50,000 · Team members 8 / 15 · Pending approvals **2** (turmeric, links to S-10).
- **Budget health:** a `BudgetThread` for every budgeted category, sorted by % used descending. This is where the stitch-in animation plays.
- **Recent expenses** table: Date, Member, Category, Description, Amount, Status. 10 rows, then "View all expenses".
- **Quick actions:** Add expense (primary), Monthly summary, Manage team.
- **Member variant:** "My month": my spent, my entries, my pending, plus a thread per category for **my** spending only. There are no team numbers.
- **Onboarding checklist** (until done): Create categories · Set budgets · Invite team.

### S-08 Expenses list — F-09
- Header: title, **Add expense**, **Import** (menu: Import CSV/Excel, Download template), **Export** (Pro+).
- **Search bar:** quick text + **Filters** button (opens FilterBuilder) + **OQL** toggle + saved filters dropdown.
- Active filters show as removable chips: `Category is Travel` `Amount ≥ ৳1,000` `Description contains "taxi"`.
- `DataTable` columns: Date, Member (hidden for Members), Category, Description, Amount (right-aligned), Status, Sync (if pending). Row actions: View, Edit (if allowed), Create adjustment.
- **Starter history notice:** "Showing the last 6 months. Upgrade to Pro to see full history."
- Empty: "No expenses match these filters. Clear filters".

#### 6.3 FilterBuilder (shared by every CRUD list)
```
┌ Filters ───────────────────────────────────────────────────────────┐
│ Match [all ▾] of the following                                      │
│  [Category ▾] [is ▾]            [Travel & Transport ▾]          ✕   │
│  [Amount   ▾] [greater or = ▾]  [1000]                          ✕   │
│  [Description▾][does not contain▾][lunch]                        ✕   │
│  ┌ Any of (OR group) ───────────────────────────────────────────┐   │
│  │ [Member ▾] [is ▾] [Fahmida Akter]                         ✕  │   │
│  │ [Member ▾] [is ▾] [Tarek Mahmud]                          ✕  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│ + Add condition   + Add group                                       │
│                                       [Clear]  [Save as…]  [Apply]  │
└─────────────────────────────────────────────────────────────────────┘
```
- The operator list depends on the field type (string / number / date / enum / bool), and both come from `GET /{resource}/search/fields`.
- In OQL mode a single-line editor with syntax highlighting is used. Invalid clauses are underlined in shapla, with a message like "Unknown field 'amout'. Did you mean 'amount'?"

### S-09 Add / Edit expense — F-03
- Form: Date* (default today, future dates disabled), Expense category* (active only), Amount (৳)*, Description.
- **Budget impact panel** (live, debounced 300 ms):
  - Within budget: paddy note "After this, Travel will be at 72% of ৳30,000."
  - Soft over: turmeric "This puts Travel over its ৳30,000 budget. It will be saved and marked Over budget."
  - Hard over: shapla "Travel is already over its ৳30,000 budget (currently ৳32,100). This category has a Hard limit, so this entry needs Manager or Admin approval." The button changes to **Submit for approval**.
- Member hint: "As a Member, you can only view and edit your own expenses."
- **Duplicate dialog:** "You saved a ৳1,200 Travel expense 'Taxi to client — Gulshan' on 15 Jun. Save this one too?" with **Save anyway** / **Cancel**.
- Offline: the button reads **Save offline**, and the toast says "Saved on this device. It will sync when you're online."
- **Edit mode** (own Saved/Pending expense, open month):
  - The impact panel calculates without the expense's old amount: "After this change, Travel will be at 103% of ৳30,000. This category has a Hard limit, so the change needs approval." The button then reads **Submit for approval**.
  - A Pending expense edited back within budget shows: "This change keeps Travel within budget. The approval request will be withdrawn." The button reads **Save changes**.
  - **Delete expense** (danger, confirm): "Delete this ৳1,200 Travel expense from 15 Jun?" For a Pending expense: "…The approval request will be cancelled."
- **Locked states** (read-only form + one action):
  - Approved/Rejected: "This expense is approved and can't be changed. Create an adjustment instead." with **Create adjustment**.
  - Closed month: "June is closed. Create an adjustment instead." with **Create adjustment**.
- **Conflict:** "This expense changed while you were editing. Nusrat approved it at 15:14." with **Reload**.

### S-10 Approvals — F-04
- Queue table: Submitted, Member, Category, Amount, Over by, Waiting time. Click a row to open the review side sheet.
- Review sheet: expense details, that category's thread for the month, and the member's month total. **Approve expense** (optional note) and **Reject expense** (reason required, min 5 chars).
- Your own expense: "You can't approve your own expense."
- The queue shows only expenses whose member is within the viewer's visibility (BR-NOT-04/05). A team-only Manager never sees other teams' requests.
- A request withdrawn or cancelled while the sheet is open shows: "Fahmida changed this expense, so it no longer needs approval." The sheet closes and the row leaves the queue.

### S-11 Adjustments — F-07
- Opened from an expense ("Create adjustment") or the list. Fields: Original expense (search), Type (Reduction / Increase), Amount*, Reason*, Date*.
- The original is shown read-only above, with a running net amount.

### S-12 Categories
- Table: Name, Status (Active/Inactive), Budget this year, Expenses count. Inline create row at the top.
- Deactivate asks "Deactivate 'Entertainment'? It disappears from the expense form. Past expenses stay in reports."
- Starter limit counter "9 of 10 categories". The 11th shows an upgrade prompt.

### S-13 Budget setup — F-06
- Header "Set yearly budget, 2026" with a year switcher (past years read-only with a lock icon).
- Explainer: "One budget per category per year. The amount is the monthly allowance. Categories without a budget are unlimited and won't alert."
- Table: Category, Budget amount (৳) (`MoneyInput`), Limit type (Soft / Hard; Hard is disabled on Starter with a "Pro feature" tooltip), Status (Set / Unlimited).
- **Save budget** and **Reset changes**. Unsaved changes warn on navigation.

### S-14 Monthly summary — F-08
- Filters: member (Pro+), month.
- KPI row: Total spent · Total budget · Remaining · Over budget (**1 category** in shapla).
- Table: Category, Budget, Spent, Remaining ("Over by ৳2,100" in shapla instead of a negative), Status chip. Total row in bold.
- Toggle **By category / By member**. The by-member view shows Member, Role chip, Total, Entries.
- **Export report** (xlsx/csv).

### S-15 Team — F-05
- Invite form: Email*, Role* (only roles the actor may assign), **Send invite**. Seat banner: "Plan: Pro. 8 of 15 seats used. You can invite 7 more members."
- Team table: Name, Email, Role chip, Status (Active/Invited/Expired/Locked), Action (Change role · Remove · Resend · Unlock).

### S-16 Permissions — F-12
```
┌ People ─────────────────┬ Permissions for Nusrat Jahan (Manager) ─────────────────────┐
│ ▾ Rahim Ahmed (Admin)   │ Granted by Rahim Ahmed · Template: Manager   [Reset to template]│
│   ▾ Nusrat Jahan (Mgr)  │                                                               │
│       Fahmida Akter     │ Expenses        View own ☑  View team ☑  View all ☑  Approve ☑ │
│       Tarek Mahmud      │ Budgets         View ☑      Manage ☐                          │
│   Karim Hasan (Member)  │ Categories      View ☑      Manage ☐                          │
│                         │ Reports         Monthly ☑   Export 🔒 (you don't hold this)    │
│ Search people…          │ Team            View ☑      Invite ☐   Assign roles ☐          │
│                         │ Import          Bulk import ☐                                  │
│                         │ Assistant       Use ☑                                           │
│                         │                                      [Discard]  [Save changes] │
└─────────────────────────┴───────────────────────────────────────────────────────────────┘
```
- Permissions the actor doesn't hold appear **locked**, with the tooltip "You can only grant permissions you have."
- Revoking shows a cascade preview: "Removing 'Approve' from Nusrat also removes it from 2 people she granted it to: Fahmida Akter, Tarek Mahmud."
- The Super Admin uses the same screen from S-21 to manage **Admins**.

### S-17 Notifications — F-04
- A grouped list with a coloured left rule by severity: shapla (approval required, over budget), turmeric (80% alert, seats), ink (new member). Each item has a title, a one-line body, a timestamp, and an action ("Review & approve").
- **Mark all as read**. Real-time arrivals slide in at the top (no other motion).

### S-18 Import wizard — F-10
Four real steps, so they're numbered:
1. **Choose file**: drag-and-drop, `.csv` / `.xlsx`, max size shown, "Download template (English | বাংলা headers)".
2. **Map columns**: auto-mapped by header name in either language, adjustable dropdowns.
3. **Validate**: a progress bar for large files (SignalR). Result: either "500 rows ready to import" or an **error grid** listing *every* problem (Row, Column, Value, Problem) with filters by column and type, plus **Download file with errors highlighted**.
4. **Import**: "500 expenses imported." With errors, the step is blocked: "Nothing was imported. Fix the 3 problems and upload again."

### S-19 Ask Outlay (assistant) — F-13
```
┌ Ask Outlay ────────────────────────────── EN|বাং ✕ ┐
│ ┌──────────────────────────────────────────────┐   │
│ │ Which team member spent the most this month? │ ◀ you
│ └──────────────────────────────────────────────┘   │
│ Fahmida Akter: ৳22,100 across 14 entries in       │
│ June 2026.                                  🔊    │
│ [mini table: top 3 members]                       │
│                                                    │
│ Try: "Are we going over Meals this month?"         │
│      "Travel expenses above ৳2,000 in May"         │
├────────────────────────────────────────────────────┤
│ [ Type a question…                       ] (🎤) ⏎ │
└────────────────────────────────────────────────────┘
```
- The mic button pulses (jamdani) while listening and shows the live transcript. It's hidden where `SpeechRecognition` is unavailable.
- Answers are spoken with `speechSynthesis` in the selected language (toggle 🔊). The voice only reads the text answer; tables are not read out.
- A refusal is styled like a normal answer, not an error: "You can ask about your own expenses. Team totals are available to Managers and Admins."

### S-20 Profile & Security
- Profile (name, phone, language, theme: System/Light/Dark), Security (change password, MFA channels, **active sessions** list with device/IP/last seen and **Sign out** per session), Offline data ("Keep offline data on this device" toggle, "Clear offline data").

### S-21 Platform overview (Super Admin)
- KPIs: Organisations, Active, On Starter / Pro / Enterprise, Plan limit reached.
- Table: Organisation, Plan chip, Users (8 / 15, NovaSoft 47 / ∞), Status (Active / Plan limit reached / Suspended), Joined. Actions: Change plan, Suspend, Manage admin permissions, Create organisation.
- **No expense amounts anywhere** on Super Admin screens.

### S-22 System (Super Admin / SQA)
- **Logs:** category select (Application, Error, Exception, Query, Audit) + date picker (dd-MM-yyyy) → preview the last 500 lines in a monospace viewer (the one place monospace is used, because it's raw log text). Slow-query lines have a shapla background and `[SLOW]` badge. **Download**.
- **Backups:** provider/engine badge (e.g. "PostgreSQL 17"), **Create backup** (OTP step-up), job list with status and **Download**.
- **Release notes:** version timeline. Each version shows New, Changed, Fixed, Known issues and a **What to test** checklist that SQA can tick locally.

### S-23 System states
- **Offline** (no cache): illustration + "You're offline and this page hasn't been saved on this device yet."
- **Service unavailable (503)**: support contact block + reference ID + **Try again**.
- **Not found (404)** and **No access (403)**: "You don't have access to Budgets. Ask your Admin for the 'Manage budgets' permission."

### S-24 Accept invitation
- "Rahim Ahmed invited you to join TechCorp BD as a Member." Fields: name, phone, password, then OTP. If the email already has an account: **Log in to accept**.

---

## 7. States Matrix (applies to every data screen)

| State | Treatment |
|---|---|
| Loading (first) | Skeleton rows matching the final layout (no spinners on full pages) |
| Loading (refetch) | 2 px jamdani progress line under the top bar; content stays |
| Empty | `EmptyState` with the action that fills it |
| Error (4xx) | Inline `ProblemAlert` near the cause |
| Error (5xx/503) | `ProblemAlert` with support contact + reference ID |
| Offline with cache | Data shown + `OfflineBanner` + "Last synced 10:42" |
| Pending sync row | Turmeric `Pending sync` chip; conflict rows show `Needs review` |
| No permission | Item hidden in the menu; direct URL → S-23 No access |

---

## 8. Localisation (UI rules)

- Library: **i18next + react-i18next + ICU** (`i18next-icu`). Namespaces per feature (`expenses.json`, `budgets.json`, …) in `src/Frontend/outlay-web/src/locales/{en,bn}/`.
- **Numbers:** a single `useFormat()` hook builds `Intl.NumberFormat` from the language:
  - EN: `new Intl.NumberFormat('en-IN', { style:'currency', currency:'BDT' })` gives "BDT 1,50,000.00". The UI replaces the currency code with the ৳ symbol and uses South-Asian grouping.
  - BN: `new Intl.NumberFormat('bn-BD', { style:'currency', currency:'BDT' })` gives "১,৫০,০০০.০০৳". Display normalises to "৳১,৫০,০০০".
- **Dates:** `Intl.DateTimeFormat(lang, …)`: "15 Jun 2026" / "১৫ জুন ২০২৬".
- **Pagination, counts, page titles** (`document.title`), toasts, validation, ProblemDetails text: all localised. Server messages arrive already localised via `Accept-Language`.
- **Input:** `MoneyInput`, `OtpInput` and search accept Bengali digits (`০-৯`) and normalise them.
- CI check: `i18next-parser` + a test that fails if any key exists in `en` but not `bn`.

---

## 9. PWA UI

- `public/manifest.json`: name "Outlay — Expense & Budget Control", short_name "Outlay", `display: standalone`, `theme_color: #1E2A5A`, `background_color: #EEF1F8`, icons 192/512 + maskable, shortcuts ("Add expense" → `/expenses/new`, "Monthly summary").
- **Install banner** (bottom sheet on mobile, top-right toast on desktop): "Install Outlay on this device for quick access and offline use." Buttons: **Install** / **Not now**.
- **Update toast:** "A new version of Outlay is ready." Button: **Refresh**.

---

## 10. Accessibility Checklist (WCAG 2.2 AA)

- Visible focus ring: 2 px jamdani outline + 2 px offset, never removed.
- Every icon-only button has an `aria-label` in the current language.
- Dialogs trap focus and return it to the trigger. Esc closes.
- Tables have `<caption>` (visually hidden) and `scope` on headers.
- Live regions: toasts `aria-live="polite"`, errors `assertive`, OTP countdown announces at 1:00 and 0:10 only.
- Minimum target size 24×24 (AA). Primary actions are 44×44.
- Tested with axe in CI (Playwright + @axe-core/playwright) for every route in both languages and both themes.

---

## 11. Scalar API Reference Theme (backend-hosted, matches the brand)

Scalar at `/scalar` uses the **"Kepler"** base theme plus `wwwroot/scalar/outlay.css` overriding `--scalar-color-accent: #5B4FD6`, `--scalar-background-1: #EEF1F8` (light) / `#121733` (dark), HTTP-method colours GET paddy, POST jamdani, PUT turmeric, DELETE shapla, and the Anek font. Title: "Outlay API", with the logo. See TRD §14.

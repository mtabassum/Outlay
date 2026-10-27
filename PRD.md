# PRD — Outlay: Expense Tracking & Budget Control (Multi-Tenant SaaS)

Version 1.0 · Requirement IDs refer to `MASTER-SPECIFICATION.md`.

## 1. Problem
Companies need a simple way to record employee spending, set budgets per category, and get warned *before* they overspend. Spreadsheets leak data between teams, have no approvals, and give no early warning.

## 2. Vision
> "Every taka accounted for, nobody surprised at month-end."

Any organisation signs up, invites its team, sets budgets and tracks expenses inside a **private workspace**, in **English or Bengali**, on **web, mobile and offline**.

## 3. Goals / Non-goals

| Goals | Non-goals (v1) |
|---|---|
| Multi-tenant isolation that is provably safe | Payment gateway / invoicing (billing is a stub) |
| 5-minute onboarding to first tracked expense | Bank-feed import, receipt OCR |
| Early budget alerts (80%) and enforced limits (hard) | Multi-currency conversion inside one org |
| Works offline and syncs later | Native mobile apps (PWA covers it) |
| Deterministic assistant (text + voice) | Free-form generative AI answers |
| Pluggable DB/ORM/cache for portability | Auto-scaling infrastructure code (compose first) |

## 4. Personas

| Persona | Example | Wants | Pain today |
|---|---|---|---|
| **Super Admin** (platform operator) | Outlay team | See orgs, plans, health; never touch customer data | Support needs without data risk |
| **Admin** | Rahim Ahmed, owner of TechCorp BD | Budget health at a glance, manage team, control permissions | Surprise overspend, no control |
| **Manager** | Nusrat Jahan, finance lead | Approve exceptions, monthly report | Chasing people for numbers |
| **Member** | Fahmida Akter, employee | Log a taxi fare in 15 seconds, even offline | Forgetting, re-typing receipts |

## 5. Key scenario (from the brief)
TechCorp BD (8 staff, Pro plan). Rahim creates five categories with budgets (Travel ৳30,000 **Hard**; Meals ৳25,000; Software ৳40,000; Office ৳20,000; Training ৳35,000 — all **Soft**), invites Nusrat (Manager) and members. Fahmida logs a ৳1,200 taxi; Travel is already over ৳30,000 and is Hard ⇒ it waits for approval; Nusrat gets a notification, approves with a note; Fahmida is notified. At month end Nusrat opens the Monthly Summary: ৳86,400 spent of ৳1,50,000; one category over budget.

## 6. Feature set & priority

| Epic | Highlights | Release |
|---|---|---|
| Workspace & onboarding | Sign-up, wizard, plans, workspace selector | R1 |
| Identity & security | OpenIddict OAuth2, BCrypt, lockout, MFA OTP, token rotation | R1–R2 |
| Team & permissions | Invitations (48 h), roles, delegated permissions UI/API | R1–R2 |
| Categories & budgets | CRUD, soft/hard limits, yearly lock | R1–R2 |
| Expenses | Add, duplicate warning, own-only visibility, adjustments | R1–R2 |
| Monitoring | 80% alert, over-budget flag, approvals, notifications | R1–R2 |
| Reports | Dashboard, Monthly Summary, member-wise, export | R1–R2 |
| Power tools | Advanced search on every list; CSV/Excel bulk import with full error report | R2 |
| Platform | DB/ORM switch, Redis/HybridCache, resilience, idempotency, rate limits, backup & log APIs, release-notes API | R1–R3 |
| Offline & PWA | Installable, IndexedDB cache, outbox sync | R4 |
| Assistant | Text + voice Q&A, guardrails | R4 |
| Quality | 5 test projects, CI/CD, Docker + load balancer, docs/ADR/C4 | R1–R5 |

## 7. User stories & acceptance criteria (representative)

| ID | Story | Acceptance (Given/When/Then) |
|---|---|---|
| US-01 | As an owner I sign up my company | Given valid org+email+password, When I submit, Then a tenant exists, I am Admin, and the wizard opens (FR-ORG-01/02) |
| US-02 | As Admin I invite a Manager | Given Pro with 8/15 seats, When I invite, Then email with 48 h link is sent; seat counter shows pending; on Starter with 3/3 I see the upgrade prompt and no invite is created (FR-PLAN-02) |
| US-03 | As Member I add an expense | Given amount 1200, today, Travel, When I save and budget is not exceeded, Then status `Saved`; Given a duplicate, Then I see a warning and can still save (FR-EXP-02) |
| US-04 | As Member I hit a Hard limit | Given Travel is Hard and over budget, When I submit, Then status `PendingApproval`, all Managers/Admins notified immediately, I see "Waiting for approval" (FR-APR-01) |
| US-05 | As Manager I approve/reject | Approve: saved + approver/time recorded + member notified. Reject: reason mandatory, entry discarded, member sees reason |
| US-06 | As Admin I get the 80% alert once | Given spend crosses 80% twice in one month, Then exactly one alert exists for that category/month (BR-NOT) |
| US-07 | As Admin I correct a mistaken expense | Original stays unchanged; adjustment row links to it; monthly net total reflects it; no negative numbers displayed (FR-EXP-04) |
| US-08 | As Admin I search any list | Filter `Description contains "taxi" AND Amount ≥ 1000 AND Status in (Saved, OverBudget)` works on every CRUD (FR-SRCH-01) |
| US-09 | As Admin I bulk-import categories from Excel | Given 200 rows with 3 errors (1 duplicate name, 2 blank), Then nothing is saved and I get all 3 errors with row/column and a downloadable error file; fixing and re-uploading succeeds (FR-BULK-01) |
| US-10 | As Member I work offline | Given no connection, I add an expense; it shows "Queued"; on reconnect it syncs once (idempotent) and then status is evaluated (FR-OFF-01) |
| US-11 | As Manager I ask "how many expenses did Tarek add today?" | Answer from my permitted scope; the same question about another company's person returns a neutral refusal (FR-ASST-01) |
| US-12 | As Super Admin I delegate | I create an Admin with a permission subset; that Admin can never exceed it nor grant what they lack (FR-AUTH-09) |
| US-13 | As user I mistype my password 3× | Account locks, I am told how to recover; audit entry written with IP (FR-AUTH-04) |
| US-14 | As QA I read what changed | `GET /api/release-notes/latest` lists features + "what to verify" (FR-OPS-03) |
| US-15 | As support I see dependency down | Redis stopped ⇒ app still works (fallback cache); DB stopped ⇒ friendly problem response with support contact (FR-DEG-01) |

## 8. Plans (reference)

| | Starter (Free) | Pro ($49/mo) | Enterprise (Custom) |
|---|---|---|---|
| Users | 3 | 15 | Unlimited |
| Categories | 10 | Unlimited | Unlimited |
| History visible | 6 months | Full | Full + archive export |
| Reports | Basic monthly | Advanced + member filter | Custom + data export |
| Alerts | In-app | + Email & SMS | + Email & SMS |
| Approval workflow / Hard limit | – | ✔ | ✔ |
| SSO / Priority support | – | – | ✔ |

## 9. Success metrics
* Time from sign-up to first expense < 5 min (median).
* ≥ 95% of over-budget events alerted within 10 s.
* 0 cross-tenant data leaks (isolation test suite green on every build).
* Monthly Summary ≤ 1 s warm.
* ≥ 90% of bulk-import errors fixed in one re-upload (error report quality).

## 10. Risks
| Risk | Mitigation |
|---|---|
| Scope is very broad (7 DB providers, offline, voice) | Release slicing in `ROADMAP.md`; provider contract tests; Access marked Windows-only |
| Cross-tenant leakage | Global query filters + mandatory tenant param in Dapper helper + automated isolation tests + cache-key prefixing |
| OTP/SMS cost & deliverability | Adapter pattern, mock in dev, Twilio fallback, rate limits |
| Offline conflicts | Idempotency keys, RowVersion, user-visible conflict UI |
| MediatR licence | A-08 |
| Voice recognition quality in Bengali varies by browser | Text always available; voice optional |

# CLAUDE.md — Outlay AI Brain

Standing instructions for any AI agent working in this repo. Short on purpose. Details live in the linked docs.

## What this is
**Outlay**: multi-tenant expense tracking & budget control SaaS. Backend .NET 10 (Clean Architecture + Vertical Slice + CQRS/MediatR, OpenIddict, RBAC). Frontend Next.js (latest) + React 19+ + Tailwind, offline-first PWA, EN/BN.
Source of truth: `MASTER-SPECIFICATION.md`. Resume point: `AI-HANDOVER.md`. Build order: `Plan.md`.

## Always
1. Read `AI-HANDOVER.md` first; end every session by updating it with the **exact next command**.
2. Work in **vertical slices**: `src/Outlay.Application/Features/<Area>/<UseCase>/` holds command/query, handler, validator, DTOs. Endpoint mapping in `Outlay.Api/Endpoints/<Area>Endpoints.cs`.
3. Layer rule: Domain ← Application ← Infrastructure ← Api. Domain has **zero** package references. Architecture tests must stay green.
4. Keep it **simple** (KISS), no speculative features (YAGNI), no copy-paste (DRY) — use the generic kits for search, import, CRUD UI.
5. Every write path: permission check + tenant scope + audit entry + idempotency (creates) + validation + EN/BN messages.
6. Every endpoint: `WithName/WithSummary/WithDescription`, response types, permission metadata, Postman request with tests.
7. Tests with the code: unit + integration (include wrong-tenant and wrong-role cases). Never "fix" a failing test by weakening it.
8. Config via `appsettings.json` **and** env (`OUTLAY_` prefix, `__` nesting). Add new settings to `.env.example` and docs.
9. Run before committing: `dotnet build && dotnet test` and in `web/`: `pnpm lint && pnpm typecheck && pnpm test`.

## Never
* Never log passwords, tokens, OTPs, secrets, connection strings, **money amounts**. Use `IOutlayLogger<T>` + redaction; don't use `Console.WriteLine`.
* Never build SQL by string concatenation with user values; never accept free-form field names (allow-list via `FilterSchema`).
* Never read or write tenant data without the tenant filter; never take tenant id from request body/query.
* Never let Super Admin endpoints return tenant expense data.
* Never store tokens in browser storage; never hard-code UI text (use message keys).
* Never enable the dev OTP (`111111`) outside Development/Testing.
* Never edit an original expense for a correction — create an adjustment.
* Never commit secrets or generated logs/backups.

## Key rules (cheat-sheet)
* Roles: SuperAdmin → Admin → Manager/Member. Grants ⊆ grantor's permissions (ceiling); revoke cascades.
* Budget: one per tenant/category/year, monthly allowance (A-01); 80% alert once/category/month; Soft = save + OverBudget + alert; Hard = PendingApproval until Manager/Admin decides; reject ⇒ discard (+ immutable ApprovalRequest).
* Plans: Starter 3 users/10 categories/6-month visible history; Pro 15 users; Enterprise unlimited. Block, don't delete.
* Login: credentials → (MFA) → login ticket → OpenIddict token. Lockout after 3 wrong passwords. OTP 5 min. BCrypt ≥12.
* Logs: `logs/<category>/dd-MM-yyyy.txt`, 5 categories, each toggleable; include class/file/method/line; exceptions include reason + fixes; slow queries red.
* Search/import apply to **every CRUD**; import = all-or-nothing + all errors reported.
* Assistant: deterministic, same permissions as UI, tenant from token, neutral refusal for anything out of scope.
* Degradation: Redis/DB/SMS down ⇒ graceful problem response with support contact from config.

## Commands
```bash
# backend
dotnet build; dotnet test; dotnet run --project src/Outlay.Api
# frontend
cd web && pnpm i && pnpm dev; pnpm lint; pnpm typecheck; pnpm test; pnpm e2e
# stack
cp .env.example .env && docker compose up --build
```

## Conventions
* C#: file-scoped namespaces, nullable on, records for DTOs/commands, `sealed` by default, async all the way with `CancellationToken`, no `async void`, small methods, meaningful names, guard clauses.
* TS: strict, no `any`, feature folders, server state in TanStack Query, forms RHF+zod.
* Naming: `<Verb><Noun>Command|Query|Handler|Validator`, tests `Method_Scenario_Expected`.
* Commits: conventional (`feat(expenses): …`). Author: `MD IKRAMUL ISLAM SIDDIQUE POROSH <poroshScientist@outlook.com>`.
* Docs to touch per feature: `RELEASENOTES.md` (Unreleased), `AI-HANDOVER.md`, Postman, OpenAPI text, message keys.

## Where things are
| Need | Look at |
|---|---|
| Requirements/rules | `MASTER-SPECIFICATION.md` |
| Tech design | `TRD.md` |
| Data model | `Schema.md` |
| How to add a CRUD | `docs/BACKEND-DEVELOPER-GUIDE.md §4`, `docs/FRONTEND-DEVELOPER-GUIDE.md §4` |
| Why we chose X | `docs/adr/` |
| Diagrams | `docs/architecture/C4-DIAGRAMS.md` |

## When unsure
Prefer the documented default (`MASTER-SPECIFICATION.md §9`), state the assumption in the PR/commit body, and add it to `AI-HANDOVER.md §3`. Do not invent new features.

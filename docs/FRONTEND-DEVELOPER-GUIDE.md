# Frontend Developer Guide

Next.js (latest stable) · React 19+ · TypeScript strict · Tailwind CSS · next-intl · TanStack Query · React Hook Form + Zod · Dexie (IndexedDB).

## 1. Structure
```
web/
  public/            manifest.json · service-worker.js · icons/ · offline.html
  src/
    app/[locale]/    App Router: (auth)/login, (app)/dashboard, expenses, … + layout.tsx (shell)
    app/api/session/ BFF route handlers (login, mfa, refresh, logout) — httpOnly cookie
    features/<name>/ api/ components/ hooks/ schemas/ pages/ index.ts      ← one folder per feature
    components/ui/   shared: DataTable, AdvancedFilterBar, BulkImportDialog, FormField, MoneyText, OtpInput, …
    lib/             http client (adds Authorization, X-Correlation-Id, Idempotency-Key, Accept-Language), auth, i18n, offline (db.ts, outbox.ts, sync.ts)
    messages/        en.json · bn.json
    styles/          tailwind theme tokens (@theme)
  tests/ e2e/        Vitest/RTL · Playwright
```
Rule: features never import from other features; share via `components/ui` or `lib`.

## 2. Data flow
Component → hook (`useExpenses`) → TanStack Query → `lib/http` → API. Writes go through `useOfflineMutation`, which tries the network, else puts the request into the **outbox** (client GUID + Idempotency-Key). Queries persist to IndexedDB so lists render offline.

## 3. i18n (English + Bengali)
* Routes `/en/...`, `/bn/...`; messages in `messages/{en,bn}.json`, namespaced per feature.
* `const t = useTranslations('expenses')`; titles via `generateMetadata` using the same keys.
* Numbers/dates/currency/pagination: `useFormatter()` / `<MoneyText>` / `<PaginationInfo>` (Bengali digits for `bn`). Never concatenate translated fragments — use ICU messages (`{from}–{to} of {total}`).
* CI test fails when key sets differ; lint rule forbids literal JSX text.

## 4. Create a new CRUD screen (recipe)
1. Backend ready? Get `/vendors`, `/vendors/search`, `/vendors/filter-schema`, import endpoints.
2. `features/vendors/schemas/vendor.ts` — Zod schema (messages are keys).
3. `features/vendors/api/vendors.ts` — typed calls (or generated from OpenAPI via `openapi-typescript`).
4. Use the kit:
   ```tsx
   export const VendorsPage = createCrudPage<Vendor>({
     resource: 'vendors', titleKey: 'vendors.title',
     columns: [{ key: 'name', labelKey: 'vendors.name', sortable: true }],
     form: VendorForm,            // RHF + zod fields
     permissions: { create: 'vendors:create', update: 'vendors:update', delete: 'vendors:delete', import: 'vendors:import' },
   });
   ```
   The kit renders `DataTable` + `AdvancedFilterBar` (fields/operators loaded from `/filter-schema`) + create/edit dialog + `BulkImportDialog` (template, upload, error grid, error CSV) + delete confirm, all localized and offline-aware.
5. Add route `app/[locale]/(app)/vendors/page.tsx` exporting the page + metadata, add nav item (auto-hidden without permission) in `lib/nav.ts`.
6. Add messages to both JSON files. 7. Add tests: schema unit test, component test, e2e happy path (EN + BN).

## 5. Styling
Tailwind v4, tokens in `styles/theme.css` (`@theme`) — see `UiSpec.md §1`. Use semantic classes (`bg-surface`, `text-muted`, `border-border`) not raw hex. Components: small, composable, accessible (Radix primitives for dialogs/menus/tooltips). Dark mode via `data-theme` + CSS variables. Respect `prefers-reduced-motion`.

## 6. Auth
Login pages call `/api/session/*` (BFF). Access token lives in memory (`AuthProvider`); refresh handled silently. Permissions fetched from `/api/v1/me/permissions` and exposed by `usePermission('expenses:approve')` / `<Can permission="…">`. UI hiding is convenience only — the API enforces.

## 7. Offline & PWA
* `public/manifest.json` + `public/service-worker.js` (hand-written, versioned cache name; precache shell/offline page; network-first navigations; cache-first static; SWR for allow-listed GETs; never cache auth).
* `lib/offline/db.ts` — Dexie schema per user+tenant (wiped on logout). `outbox.ts` — queue; `sync.ts` — replays on `online`, focus, and app start (Background Sync when available) with backoff.
* Install prompt: `useInstallPrompt()` captures `beforeinstallprompt`; iOS hint card.
* Test offline in Chrome DevTools → Network → Offline; Application → Service Workers.

## 8. Assistant (text + voice)
`features/assistant`: `useSpeechRecognition(lang)` and `useSpeechSynthesis()` wrap Web Speech API with feature detection; `POST /api/v1/assistant/query`. Always keep the text box usable.

## 9. Error & degradation UX
`lib/http` maps ProblemDetails → `AppError{code, message, support}`. `<ErrorBoundary>` + toast show localized text; `code` starting `DEP_` shows `SupportContactCard` (env-configured name/phone/email). Network failure ⇒ offline banner, queue writes.

## 10. Testing
`pnpm test` (Vitest + React Testing Library), `pnpm e2e` (Playwright: EN/BN, mobile viewport, offline, axe a11y). Mock API with MSW. Snapshot tests only for stable UI atoms.

## 11. Upgrading Next.js/React/Tailwind
Bump in a branch, run `pnpm dlx @next/codemod@latest` where applicable, fix typing, run all tests + e2e, test PWA install/offline. Avoid experimental flags and deprecated APIs; weekly canary CI against `@latest`.

## 12. Definition of done
Both languages · dark/light · 360 px · keyboard · loading/empty/error/offline/forbidden states · shared components used · tests · no hard-coded text.

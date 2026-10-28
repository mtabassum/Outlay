# Outlay — Frontend Developer Guide

| Field | Value |
|---|---|
| Audience | React / TypeScript developers (and AI agents) working on `src/Frontend/outlay-web` |
| Version | 0.0.0 · 2026-10-28 |
| Stack | React 19 · TypeScript 6 (strict) · Vite 8 · Tailwind CSS v4 · React Router 7 · TanStack Query 5 / Table · Zustand · React Hook Form + Zod · Dexie · i18next · vite-plugin-pwa |
| Read first | [CLAUDE.md](../../CLAUDE.md) · [UiSpec.md](../UiSpec.md) (design) · [AppFlow.md](../AppFlow.md) (flows) · [TRD.md §12](../TRD.md) (architecture) |

> This guide is the practical "how". When it disagrees with UiSpec (look) or TRD (architecture), they win: fix this guide in the same PR.
> The repo currently holds the Vite + React template. The structure below is what milestone M0 builds.

---

## Contents

1. [Getting started](#1-getting-started)
2. [Project structure](#2-project-structure)
3. [Design system & Tailwind tokens](#3-design-system--tailwind-tokens)
4. [Talking to the API](#4-talking-to-the-api)
5. [Add a CRUD screen: worked example (Categories)](#5-add-a-crud-screen-worked-example)
6. [Offline-first: reads, writes and sync](#6-offline-first-reads-writes-and-sync)
7. [Localisation (English / বাংলা)](#7-localisation-english--বাংলা)
8. [Auth, permissions and real-time](#8-auth-permissions-and-real-time)
9. [PWA](#9-pwa)
10. [Assistant (voice & text)](#10-assistant-voice--text)
11. [Errors and the support contact](#11-errors-and-the-support-contact)
12. [Testing](#12-testing)
13. [Upgrades](#13-upgrades)
14. [Pull-request checklist](#14-pull-request-checklist)

---

## 1. Getting started

### 1.1 Prerequisites
- **Node 24 LTS** and npm (the version is in `.nvmrc`).
- The backend running locally (`dotnet run --project src/Backend/Outlay.Api --launch-profile https`). See the Backend Developer Guide §1.

### 1.2 Run

```bash
cd src/Frontend/outlay-web
npm ci
npm run dev                  # http://localhost:5173 (default; set VITE_DEV_PORT to change it, and Vite moves to the next free port if busy)
```

`vite.config.ts` proxies `/api`, `/connect`, `/hubs` and `/openapi` to the API, so the browser sees one origin, cookies work and there's no CORS in development. **No port is hard-coded.** The API address and the dev port come from environment variables, so each teammate can use their own:

```ts
// vite.config.ts (after M0)
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import { VitePWA } from 'vite-plugin-pwa'
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')           // reads .env.development + .env.development.local
  const api = env.VITE_API_PROXY_TARGET                    // e.g. https://localhost:<your API port>
  if (!api) throw new Error('Set VITE_API_PROXY_TARGET in .env.development.local (see Frontend Developer Guide §1.4)')
  const proxied = { target: api, secure: false, changeOrigin: true }

  return {
    plugins: [
      react(),
      tailwindcss(),
      VitePWA({ strategies: 'injectManifest', srcDir: 'src', filename: 'service-worker.ts', manifest: false,
                injectRegister: false, devOptions: { enabled: true, type: 'module' } }),
    ],
    server: {
      port: Number(env.VITE_DEV_PORT) || 5173,             // Vite picks the next free port if this one is busy
      proxy: { '/api': proxied, '/connect': proxied, '/openapi': proxied, '/hubs': { ...proxied, ws: true } },
    },
  }
})
```

Log in with the demo accounts (Backend guide §1.2). In development the OTP is **111111**, and a ribbon on the login screen reminds you.

### 1.3 Scripts

| Script | Does |
|---|---|
| `npm run dev` | Vite dev server |
| `npm run build` | `tsc -b && vite build` (type errors fail the build) |
| `npm run lint` / `npm run format` | ESLint 10 / Prettier |
| `npm run typecheck` | `tsc -b --noEmit` |
| `npm test` / `npm run test:watch` | Vitest |
| `npm run test:e2e` | Playwright (starts the dev server) |
| `npm run gen:api` | Regenerates `src/api-types.ts` from `https://localhost:7024/openapi/v1.json` by default (taken from `VITE_API_PROXY_TARGET`; the API must be running) |
| `npm run gen:permissions` | Regenerates `src/shared/lib/permissions.ts` from the backend catalogue |
| `npm run i18n:check` | Fails if any key is missing in `en` or `bn` |

### 1.4 Environment variables

`.env.development` (committed, shared defaults, no secrets) and `.env.production`:

```dotenv
VITE_API_BASE_URL=/api                       # relative: works behind the Vite proxy and the nginx load balancer on any port
VITE_API_PROXY_TARGET=https://localhost:7024 # shared default = the API's https port in launchSettings.json
VITE_DEV_PORT=5173
VITE_SUPPORT_NAME=Outlay Team
VITE_SUPPORT_PHONE=+880 1XXX-XXXXXX
VITE_SUPPORT_EMAIL=support@outlay.app
```

**Different ports on your machine?** Create `.env.development.local` next to it. It's git-ignored (`.env.*.local` in the repo's `.gitignore`), and Vite loads it on top of `.env.development`:

```dotenv
# .env.development.local (yours only, never committed)
VITE_API_PROXY_TARGET=https://localhost:<port>  # whatever your API printed as "Now listening on"
VITE_DEV_PORT=3000
```

Because `VITE_API_BASE_URL` is relative, no port ever appears in application code.

`VITE_*` values are compiled into the public bundle. **Never put a secret there.**

---

## 2. Project structure

```
src/Frontend/outlay-web/
├── public/                      manifest.json, icons/ (192, 512, maskable), robots.txt
├── src/
│   ├── main.tsx                 bootstraps i18n, registers the service worker, renders <App/>
│   ├── app/
│   │   ├── App.tsx              <Providers><RouterProvider/></Providers>
│   │   ├── providers.tsx        QueryClient, I18nextProvider, ThemeProvider, AuthProvider, SyncProvider, SignalRProvider
│   │   ├── router.tsx           route table (lazy routes, permission guards)
│   │   └── layouts/             AppShell (rail, top bar, mobile tab bar), AuthLayout
│   ├── features/
│   │   └── categories/          ← one folder per feature (mirrors backend Features/)
│   │       ├── api.ts           typed calls (uses api-types.ts)
│   │       ├── queries.ts       TanStack Query hooks + query keys
│   │       ├── schema.ts        Zod schema(s) matching the backend validator
│   │       ├── components/      CategoryForm.tsx, CategoryRowActions.tsx
│   │       ├── pages/           CategoriesPage.tsx
│   │       └── index.ts         public exports (routes)
│   ├── shared/
│   │   ├── ui/                  design-system components (UiSpec §4): Button, MoneyInput, DataTable, BudgetThread…
│   │   ├── lib/                 http.ts, format.ts, problem.ts, permissions.ts (generated), ids.ts (uuidv7)
│   │   ├── hooks/               useFormat, usePermission, useOnlineStatus, useCountdown, usePageTitle
│   │   └── search/              FilterBuilder, OqlEditor, oql-parser.ts, useSearchState
│   ├── offline/                 db.ts (Dexie), outbox.ts, sync-engine.ts, useOfflineMutation.ts, conflict.ts
│   ├── locales/{en,bn}/         common.json, categories.json, expenses.json, …
│   ├── styles/                  theme.css (@theme tokens), fonts.css
│   ├── service-worker.ts        built to service-worker.js
│   └── api-types.ts             GENERATED. Never edit by hand.
└── tests/e2e/                   Playwright specs
```

**Rules**
- Features may import from `shared/` and `offline/`, **never from another feature's internals** (only through its `index.ts`). ESLint `import/no-restricted-paths` enforces it.
- No component outside `shared/ui` defines its own button, input or table styles.

---

## 3. Design system & Tailwind tokens

Tailwind v4 is configured **in CSS**, so there's no `tailwind.config.js`:

```css
/* src/styles/theme.css */
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));

@theme {
  --color-ink: #1E2A5A;      --color-ink-soft: #4A5683;
  --color-paper: #EEF1F8;    --color-surface: #FFFFFF;   --color-line: #D5DBEA;
  --color-paddy: #2F8F5B;    --color-turmeric: #E0A019;  --color-shapla: #C92A4B;  --color-jamdani: #5B4FD6;
  --font-sans: "Anek Latin Variable", "Noto Sans", system-ui, sans-serif;
  --radius-input: 8px;  --radius-panel: 14px;
}

.dark {
  --color-ink: #C9D2F5;      --color-ink-soft: #9AA5D1;
  --color-paper: #121733;    --color-surface: #1A2045;   --color-line: #2C3463;
  --color-paddy: #5CC48A;    --color-turmeric: #F2BE4A;  --color-shapla: #F06C86;  --color-jamdani: #9B92FF;
}

:lang(bn) { font-family: "Anek Bangla Variable", "Noto Sans Bengali", sans-serif; }
.tabular  { font-variant-numeric: tabular-nums; }
```

Tailwind generates utilities from the token namespaces: `bg-paper`, `text-ink`, `border-line`, `text-shapla`, `rounded-panel`, `rounded-input`, and so on. When you need a raw variable, v4 syntax is `bg-(--color-ink)`.

- **No hex colours or arbitrary colour values in components.** An ESLint rule flags `#xxxxxx` and `[color:…]` in `.tsx`.
- Status colours always come **with a label or icon** (UiSpec §2.1).
- Copy rules: sentence case, verbs on buttons, no ALL-CAPS labels (UiSpec §5).
- The signature component `BudgetThread` lives in `shared/ui/BudgetThread.tsx`. Use it for any budget-progress display, and don't build a second progress bar.

---

## 4. Talking to the API

### 4.1 Generated types

`npm run gen:api` uses `openapi-typescript` to produce `src/api-types.ts`. Import types from it so frontend and backend can't drift:

```ts
import type { components } from '@/api-types'
export type CategoryDto = components['schemas']['CategoryDto']
```

### 4.2 The HTTP client (`shared/lib/http.ts`)

One thin `fetch` wrapper does everything cross-cutting. Don't call `fetch` directly in features.

- Adds `Authorization: Bearer <in-memory access token>`, `Accept-Language` (current i18n language), `X-Correlation-Id` (uuidv7).
- Adds `Idempotency-Key` to POST/PUT/PATCH/DELETE (or reuses the one the outbox passes in).
- On 401, refreshes once (refresh token in the HttpOnly cookie) and retries. If that fails, it logs out.
- Parses non-2xx into a typed `ProblemError` (RFC 9457 ProblemDetails, with `code`, `errors`, `correlationId`, `support`).
- Classifies network failures and 5xx/503 as `isOfflineLike`, which tells the offline layer to take over.

```ts
export const categoriesApi = {
  get:    (id: string)              => http.get<CategoryDto>(`/v1/categories/${id}`),
  search: (req: SearchRequest)      => http.post<PagedResult<CategoryDto>>('/v1/categories/search', req),
  create: (body: CreateCategory, opts?: RequestOpts) => http.post<CategoryDto>('/v1/categories', body, opts),
  update: (id: string, body: UpdateCategory, opts?: RequestOpts) => http.put<CategoryDto>(`/v1/categories/${id}`, body, opts),
  deactivate: (id: string, opts?: RequestOpts) => http.post<void>(`/v1/categories/${id}/deactivate`, undefined, opts),
}
```

### 4.3 Query keys

Keys always include **tenant and language**, so switching workspace or language never shows stale data:

```ts
export const categoryKeys = {
  all:    (t: string, lng: string) => ['categories', t, lng] as const,
  search: (t: string, lng: string, req: SearchRequest) => [...categoryKeys.all(t, lng), 'search', req] as const,
  detail: (t: string, lng: string, id: string) => [...categoryKeys.all(t, lng), 'detail', id] as const,
}
```

`useScopeKey()` returns `{ tenantId, lng }` for building keys.

---

## 5. Add a CRUD screen: worked example

We'll build the **Categories** screen (UiSpec S-12) on top of the backend slices from the Backend guide §3.

### Step 1: Schema (mirror the backend validator)

```ts
// features/categories/schema.ts
import { z } from 'zod'

export const categoryFormSchema = z.object({
  name: z.string().trim().min(1, 'categories:errors.nameRequired').max(50, 'categories:errors.nameTooLong'),
  description: z.string().trim().max(250).optional(),
})
export type CategoryForm = z.infer<typeof categoryFormSchema>
```

Messages are **i18n keys**. `FormField` translates them, so the same schema works in both languages.

### Step 2: Queries and mutations

```ts
// features/categories/queries.ts
export function useCategorySearch(req: SearchRequest) {
  const { tenantId, lng } = useScopeKey()
  return useOfflineQuery({
    queryKey: categoryKeys.search(tenantId, lng, req),
    queryFn: () => categoriesApi.search(req),
    offline: { table: 'categories', search: req },     // fallback: filter the IndexedDB copy locally
  })
}

export function useCreateCategory() {
  const { tenantId, lng } = useScopeKey()
  return useOfflineMutation({
    resource: 'categories',
    mutationFn: (body: CategoryForm, ctx) => categoriesApi.create(body, { idempotencyKey: ctx.idempotencyKey }),
    optimistic: (body, ctx) => ({ id: ctx.tempId, ...body, isActive: true }),     // shown with "Pending sync" if offline
    invalidate: [categoryKeys.all(tenantId, lng)],
  })
}
```

`useOfflineMutation` runs the request when online. When offline (or on an offline-like failure) it writes to the outbox and the optimistic record instead (§6).

### Step 3: The form

```tsx
// features/categories/components/CategoryForm.tsx
export function CategoryForm({ onDone }: { onDone: () => void }) {
  const { t } = useTranslation('categories')
  const form = useForm<CategoryForm>({ resolver: zodResolver(categoryFormSchema) })
  const create = useCreateCategory()

  const submit = form.handleSubmit(async values => {
    const result = await create.mutateAsync(values)
    toast.success(result.queued ? t('toast.savedOffline') : t('toast.created'))   // "Category added"
    onDone()
  })

  return (
    <form onSubmit={submit} className="grid gap-4">
      <FormField form={form} name="name" label={t('fields.name')} autoFocus />
      <FormField form={form} name="description" label={t('fields.description')} />
      <ProblemAlert error={create.error} form={form} />   {/* maps server field errors onto inputs */}
      <Button type="submit" loading={create.isPending}>{t('actions.add')}</Button>   {/* "Add category" */}
    </form>
  )
}
```

### Step 4: The page (list + search + import + permissions)

```tsx
// features/categories/pages/CategoriesPage.tsx
export default function CategoriesPage() {
  const { t } = useTranslation('categories')
  usePageTitle(t('title'))                                  // document.title in the current language
  const search = useSearchState({ resource: 'categories', defaultSort: [{ field: 'name', dir: 'asc' }] })
  const { data, isLoading, isOfflineData } = useCategorySearch(search.request)
  const canManage = usePermission(P.categories.manage)

  return (
    <Page
      title={t('title')}
      actions={canManage && (<>
        <ImportMenu resource="categories" />                {/* template download + ImportWizard (S-18) */}
        <Dialog trigger={<Button>{t('actions.add')}</Button>}>{close => <CategoryForm onDone={close} />}</Dialog>
      </>)}>
      <SearchBar state={search} resource="categories" />   {/* quick text + FilterBuilder + OQL + saved filters */}
      {isOfflineData && <OfflineBanner />}
      <DataTable
        columns={useCategoryColumns()}
        data={data?.items}
        total={data?.totalCount}
        loading={isLoading}
        state={search}                                       {/* server-side sort + pagination */}
        empty={<EmptyState text={t('empty')} action={canManage && t('actions.add')} />}
      />
      <PlanLimitHint resource="categories" />               {/* "9 of 10 categories" on Starter */}
    </Page>
  )
}
```

`FilterBuilder` gets its fields and operators from `GET /api/v1/categories/search/fields`, so you don't write any filter UI per resource.

### Step 5: Route and menu

```ts
// features/categories/index.ts
export const categoryRoutes: AppRoute[] = [
  { path: 'categories', lazy: () => import('./pages/CategoriesPage'), permission: P.categories.read,
    nav: { labelKey: 'nav.categories', icon: 'Tags', order: 60 } },
]
```

`router.tsx` collects feature routes. `RequirePermission` guards the route (403 page S-23), and the rail and mobile menus render only items the user may see.

### Step 6: Strings in both languages

```json
// locales/en/categories.json
{ "title": "Categories", "fields": { "name": "Name", "description": "Description" },
  "actions": { "add": "Add category", "deactivate": "Deactivate" },
  "toast": { "created": "Category added", "savedOffline": "Saved on this device. It will sync when you're online." },
  "empty": "No categories yet. Add your first category.",
  "errors": { "nameRequired": "Enter a category name.", "nameTooLong": "Use 50 characters or fewer." } }
```

```json
// locales/bn/categories.json
{ "title": "খরচের খাত", "fields": { "name": "নাম", "description": "বিবরণ" },
  "actions": { "add": "খাত যোগ করুন", "deactivate": "নিষ্ক্রিয় করুন" },
  "toast": { "created": "খাত যোগ হয়েছে", "savedOffline": "এই ডিভাইসে সংরক্ষিত। অনলাইনে এলে সিঙ্ক হবে।" },
  "empty": "এখনও কোনো খাত নেই। প্রথম খাত যোগ করুন।",
  "errors": { "nameRequired": "খাতের নাম লিখুন।", "nameTooLong": "৫০ অক্ষরের মধ্যে লিখুন।" } }
```

Bengali copy should be reviewed by a native speaker before release (tracked in the release SQA checklist).

### Step 7: Offline storage registration

Add the table to Dexie (bump the version) and register the resource with the sync engine:

```ts
// offline/db.ts
this.version(3).stores({
  categories: 'id, normalizedName, isActive, changeVersion',
  // …existing tables unchanged
})
// offline/resources.ts
registerSyncResource({ name: 'categories', table: 'categories', pullKey: 'categories' })
```

### Step 8: Tests

```tsx
// features/categories/pages/CategoriesPage.test.tsx  (Vitest + RTL + MSW)
it('shows server duplicate error on the name field', async () => {
  server.use(http.post('/api/v1/categories', () =>
    HttpResponse.json({ status: 409, code: 'category.duplicate_name', title: 'This category already exists.' }, { status: 409 })))
  renderWithProviders(<CategoriesPage />, { user: personas.admin })
  await userEvent.click(screen.getByRole('button', { name: 'Add category' }))
  await userEvent.type(screen.getByLabelText('Name'), 'Travel')
  await userEvent.click(screen.getByRole('button', { name: 'Add category' }))
  expect(await screen.findByText('This category already exists.')).toBeVisible()
})

it('hides Add for members', () => {
  renderWithProviders(<CategoriesPage />, { user: personas.member })
  expect(screen.queryByRole('button', { name: 'Add category' })).toBeNull()
})
```

```ts
// tests/e2e/categories.spec.ts (Playwright): one happy path per role, in both languages
test('admin adds a category offline and it syncs once', async ({ page, context }) => {
  await loginAs(page, 'admin')
  await page.goto('/categories')
  await context.setOffline(true)
  await addCategory(page, 'Courier')
  await expect(page.getByText('Pending sync')).toBeVisible()
  await context.setOffline(false)
  await expect(page.getByText('Pending sync')).toBeHidden()
  expect(await countCategoriesViaApi('Courier')).toBe(1)          // idempotency: exactly one record
})
```

---

## 6. Offline-first: reads, writes and sync

```
read:  network ──ok──► TanStack cache + write-through to IndexedDB
          └─fail/5xx──► IndexedDB (isOfflineData = true)

write: online ──► API (Idempotency-Key) ──► invalidate queries
       offline ─► outbox (Idempotency-Key, baseRowVersion) + optimistic row (syncStatus: 'pending')
                    └─► sync engine replays FIFO when online → 2xx: swap temp id / 409: conflict / 422,403: rejected
pull:  GET /api/v1/sync/changes?since=<cursor> → upsert rows, apply soft-deletes, store new cursor
```

- **Use `useOfflineQuery` / `useOfflineMutation`** for any resource that should work offline (expenses, categories, budgets, notifications). Plain `useQuery`/`useMutation` are fine for online-only screens (System, Platform).
- The outbox item keeps its **Idempotency-Key for its whole life**, so a timeout-then-retry never creates two records.
- Business rules (budget limits, approvals) are decided by the **server** at sync time. The UI says "Saved on this device" and later shows the real outcome ("Needs approval", "Rejected").
- Conflicts surface in the sync panel (`SyncCenter`), where the user picks **Keep mine** (resubmit with the latest `rowVersion`) or **Use server**.
- **Never** store tokens in IndexedDB. Logout calls `offlineDb.clearForUser()` unless the user chose "Keep offline data on this device".

---

## 7. Localisation (English / বাংলা)

- `const { t } = useTranslation('<namespace>')`. Namespaces are lazy-loaded per feature. The `bn` bundles are precached by the service worker.
- **ICU messages** for plurals and variables: `"members": "{count, plural, one {# member} other {# members}}"`. Never concatenate strings.
- **Numbers, money, dates, percentages and pagination:** only through `useFormat()`:

```ts
const f = useFormat()
f.money(150000)            // en → "৳1,50,000"   bn → "৳১,৫০,০০০"
f.number(67)               // en → "67"          bn → "৬৭"
f.date('2026-06-15')       // en → "15 Jun 2026" bn → "১৫ জুন ২০২৬"
f.percent(0.82)            // en → "82%"         bn → "৮২%"
f.pageInfo(21, 40, 67)     // en → "Showing 21–40 of 67"  bn → "৬৭টির মধ্যে ২১–৪০ দেখানো হচ্ছে"
```

- Page titles go through `usePageTitle(t('title'))`, which updates `document.title` when the language changes.
- Inputs accept Bengali digits. `parseLocalizedNumber('১২০০')` gives `1200`. `MoneyInput` and `OtpInput` already do this.
- Changing language: `setLanguage('bn')` updates i18next and `<html lang>`, saves the preference (`PATCH /api/v1/me/preferences`), and refetches queries that contain server text.
- CI runs `npm run i18n:check`. A missing key in either language fails the build.

---

## 8. Auth, permissions and real-time

- `AuthProvider` holds the **access token in memory** (never in localStorage or IndexedDB). The refresh token is an HttpOnly cookie the JS can't read. Silent refresh runs 60 s before expiry.
- Login is two steps: `password` grant → `mfa_required` → `urn:outlay:otp` grant. `useLogin()` returns a state machine (`credentials → otp → workspace → done`) that the pages S-01/S-02/S-06 render. The OTP screen uses `useCountdown(expiresAt)` for the 05:00 ring and `resendAvailableAt` for the 60 s lock.
- The web UI **never sends `session_duration`** (it's an API-only option).
- Permissions come from `GET /api/v1/me/permissions`. Use:

```tsx
const canApprove = usePermission(P.expenses.approve)
<Can permission={P.reports.export}><ExportButton /></Can>
```

  Hiding UI is for convenience only. The API enforces every permission.
- **SignalR** (`SignalRProvider`, `@microsoft/signalr`, WebSockets transport): subscribe with `useHubEvent('notificationCreated', handler)`. Events: `notificationCreated`, `importProgress`, `permissionsChanged` (refetch permissions), `expenseStatusChanged`.

---

## 9. PWA

- **Manifest:** `public/manifest.json` (UiSpec §9), linked in `index.html` with `<link rel="manifest" href="/manifest.json">`, `<meta name="theme-color" content="#1E2A5A">` and Apple touch icons.
- **Service worker:** `src/service-worker.ts` (Workbox via `injectManifest`) builds to **`/service-worker.js`**:
  - precaches the app shell, fonts and `bn`/`en` locale bundles,
  - `NetworkFirst` (3 s timeout) for `GET /api/v1/**` as a second safety net,
  - Background Sync tag `outlay-outbox` to flush the outbox,
  - listens for `SKIP_WAITING` to activate updates.
- **Install prompt:** `useInstallPrompt()` captures `beforeinstallprompt` (Chromium) and shows `InstallPrompt` after the second visit or the first saved expense. iOS gets the "Share → Add to Home Screen" hint.
- **Updates:** `useServiceWorkerUpdate()` shows "A new version of Outlay is ready" with a **Refresh** button.
- **Test it:** `npm run build && npm run preview`, then Chrome DevTools → Application (Manifest, Service workers, IndexedDB) and Lighthouse → PWA. Lighthouse CI runs in the pipeline.
- The nginx web image serves `index.html`, `manifest.json` and `service-worker.js` with `Cache-Control: no-cache`, so updates are picked up.

---

## 10. Assistant (voice & text)

- `AssistantPanel` (S-19) posts `{ text, locale }` to `/api/v1/assistant/ask` and renders `answerText`, optional `data` (mini table) and `suggestions`.
- `useSpeechInput(lang)` wraps `SpeechRecognition`/`webkitSpeechRecognition` (`en-US` or `bn-BD`, interim results). It returns `{ supported, listening, transcript, start, stop }`. When `supported` is false, hide the mic button and keep text input.
- `useSpeak()` wraps `speechSynthesis`. It picks a voice matching the language if one is installed and otherwise stays silent (the text is always shown).
- Show the one-time privacy note before the first mic use: the browser's speech service may process audio in the cloud.
- Never construct answers on the client. The server is the only source, so guardrails always apply.

---

## 11. Errors and the support contact

- `ProblemAlert` renders any `ProblemError`: a localised title, field errors (mapped onto form fields when a `form` prop is passed), and the **reference ID** (`correlationId`) to quote to support.
- For `503 dependency_unavailable` and other 5xx, it also shows **the support contact**: "Contact {name} at {phone} or {email}". The contact comes from `GET /api/v1/system/support-contact`, cached in IndexedDB, with a fallback to the `VITE_SUPPORT_*` variables, so it works even when the API is unreachable.
- `429` shows the `Retry-After` countdown.
- A root `ErrorBoundary` catches render errors, shows S-23 with the reference ID, and reports to the console in dev.

---

## 12. Testing

| Layer | Tool | What to test |
|---|---|---|
| Unit | Vitest | `format.ts` (both languages), `oql-parser.ts` (shares test vectors with the backend parser), outbox ordering, `useCountdown` |
| Component | Vitest + React Testing Library + MSW | Forms (client and server errors), permission-gated UI, empty, error and offline states |
| E2E | Playwright | Critical flow per role (login + OTP 111111, add expense soft/hard limit, approve, import with errors, language switch, offline create → sync once) |
| Accessibility | @axe-core/playwright | Every route in EN and BN, light and dark |
| PWA / performance | Lighthouse CI | PWA installable, LCP budget, a11y score |

Query elements by role and accessible name (`getByRole('button', { name: 'Add category' })`), not by CSS classes or test ids, so tests also check accessibility. Shared fixtures (`personas`, `renderWithProviders`, MSW handlers generated from OpenAPI examples) live in `src/test/`.

---

## 13. Upgrades

- **Renovate** groups React, Vite, Tailwind and TanStack updates. Patch updates auto-merge on green CI. Read the changelog for minor and major updates.
- Keep the code upgrade-friendly: function components only, no deprecated React APIs (`forwardRef` isn't needed in React 19, since `ref` is a prop), no imports from libraries' internal paths, and Tailwind tokens only in `theme.css`.
- After a major upgrade: `npm run typecheck && npm test && npm run test:e2e && npm run build`, a Lighthouse run, and a RELEASENOTES entry with SQA checks for the PWA and offline flows.

---

## 14. Pull-request checklist

- [ ] Feature code lives in `features/<feature>/`. Shared pieces come from `shared/ui`, and nothing duplicates them.
- [ ] Types come from `api-types.ts`. No hand-written copies of DTOs.
- [ ] All calls go through `http.ts`. Mutations that should work offline use `useOfflineMutation`.
- [ ] Every string is an i18n key in **en and bn**. Numbers, dates and money use `useFormat()`. `usePageTitle` is set.
- [ ] No hex colours. Tokens only. Status shown with text or icon, not colour alone.
- [ ] Permission-gated actions use `usePermission`/`<Can>`. The route has `permission`.
- [ ] Loading (skeleton), empty, error, offline and pending-sync states are handled (UiSpec §7).
- [ ] Keyboard and screen reader: labels, focus ring, dialogs return focus.
- [ ] Vitest + RTL tests. Playwright updated for new flows. axe passes.
- [ ] RELEASENOTES `[Unreleased]` + SQA rows. AI-HANDOVER updated.

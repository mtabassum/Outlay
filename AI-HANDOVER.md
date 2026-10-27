# AI-HANDOVER

> Read this first when you resume. Update it at the end of **every** work session. Keep it short and exact.

## 1. Snapshot
| Item | Value |
|---|---|
| Project | Outlay — multi-tenant Expense Tracking & Budget Control |
| Repo | https://github.com/mtabassum/Outlay (**contents unverified**; fetch failed when docs were authored) |
| Phase | Documentation baseline complete (0.0.1). **No application code written.** |
| Current milestone | M0 — not started |
| Last green build | n/a |
| Branch | `main` (create `feat/m0-scaffold` for M0) |
| Author identity for commits | MD IKRAMUL ISLAM SIDDIQUE POROSH `<poroshScientist@outlook.com>` |

## 2. What exists
`PRD.md AppFlow.md UiSpec.md TRD.md Schema.md Plan.md MASTER-SPECIFICATION.md MASTER-PROMPT.md ROADMAP.md RELEASENOTES.md AI-HANDOVER.md CLAUDE.md README.md docs/{BACKEND-DEVELOPER-GUIDE.md, FRONTEND-DEVELOPER-GUIDE.md, adr/*, architecture/C4-DIAGRAMS.md}`

## 3. Decisions awaiting the owner (defaults apply until answered)
See `MASTER-SPECIFICATION.md §9 (A-01…A-11)` and `§10 (Q-01…Q-04)`. Most impactful: **A-01** (monthly vs yearly budget meaning), **A-02** (Hard limit Pro-only), **A-04** (SMS gateway), **A-08** (MediatR licence/version).

## 4. Exact next command
```bash
# 1) Verify the repo, then add the docs at its root
git clone https://github.com/mtabassum/Outlay.git && cd Outlay && ls -la
# 2) Create the working branch and start Milestone M0 (see Plan.md)
git checkout -b feat/m0-scaffold
git config user.name  "MD IKRAMUL ISLAM SIDDIQUE POROSH"
git config user.email "poroshScientist@outlook.com"
dotnet --list-sdks            # expect a .NET 10 SDK
node -v && corepack enable    # expect current LTS Node; pnpm via corepack
dotnet new sln -n Outlay
```
Then follow `Plan.md → M0` task by task.

## 5. M0 checklist (first work)
- [ ] `global.json`, `Directory.Build.props`, `Directory.Packages.props`, `.editorconfig`, `.gitignore`, `.dockerignore`
- [ ] Projects: `Outlay.Domain/Application/Infrastructure/Api` + `tests/*` (6 projects) added to solution
- [ ] `web/` Next.js app (TypeScript, Tailwind, next-intl, ESLint, Vitest)
- [ ] `.env.example`, `compose.yml` skeleton, `.github/workflows/ci.yml` skeleton
- [ ] Architecture tests (layer rules) green
- [ ] First commit: `chore: scaffold solution and web app`

## 6. Gotchas to remember
* InMemory ⇒ single API replica (A-07). Access ⇒ Windows only (A-06).
* Dev OTP `111111` only works in Development/Testing; startup must fail if enabled in Production.
* Never log money amounts, tokens, OTPs, passwords. Add a test whenever you add a log statement near sensitive data.
* Tenant isolation test suite must run on every provider.
* Keep `RELEASENOTES.md` heading format — the API parses it.

## 7. Session log (append newest on top)
| Date | Who | Done | Next |
|---|---|---|---|
| 2026-10-28 | Docs author | Produced documentation set (0.0.1) | M0 scaffolding |

## 8. If you are low on budget/context
Finish current slice to green → update §1, §4, §7 here + `RELEASENOTES.md` + `ROADMAP.md` → commit → stop. The next agent must be able to continue from §4 alone.

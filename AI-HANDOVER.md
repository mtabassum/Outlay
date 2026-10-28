# AI Handover — Outlay

> **Purpose:** whoever picks up Outlay next (a human or an AI session) reads this first, after `CLAUDE.md`.
> **Rule:** update this file at the end of every working session. Keep it short and current, and delete stale lines.

| Field | Value |
|---|---|
| Last updated | 2026-10-28 |
| Updated by | AI session |
| Current version | v0.0.0: specification baseline (no application code yet) |
| Current milestone | **M0 — Foundations** (not started, apart from the repo inspection in M0-01) |
| Next task | **M0-01** (remaining part): push the WeatherForecast deletion, remove `AddControllers()`/`MapControllers()` and the Vite template content, then **M0-02** build config |

---

## 1. Current State

### Done
- **Specification set v0.0.0**, all files in this zip/repo:
  - Core specs: `docs/PRD.md`, `docs/AppFlow.md`, `docs/UiSpec.md`, `docs/TRD.md`, `docs/Schema.md`, `docs/Plan.md`
  - Architecture: `docs/adr/0001…0018`, `docs/diagrams/C4-DIAGRAMS.md`
  - Guides: `docs/guides/BACKEND-DEVELOPER-GUIDE.md`, `docs/guides/FRONTEND-DEVELOPER-GUIDE.md` (they show the *intended* code shape; verify snippets as code lands)
  - Governance: `CLAUDE.md`, `MASTER-SPECIFICATION.md` (incl. §7 limitations and deviations), `MASTER-PROMPT.md`, `ROADMAP.md`, `RELEASENOTES.md`, this file
- **Sources:** the training brief `Requirement.pdf` (*Expense Tracking & Budget Control System — Multi-Tenant SaaS, OOP Training Program, Project 3*), read in full, plus the owner's extended engineering brief. Every item is mapped in `MASTER-SPECIFICATION.md §4`.
- **Repo inspected** (`mtabassum/Outlay`, commit `b5fc8f3`, 34 files), a fresh scaffold:
  - Root `Outlay.slnx`. `src/Backend/Outlay.{Domain, Application, Infrastructure, Api}` on `net10.0` with correct Clean Architecture references.
  - `Outlay.Api` was the template. The owner deleted the WeatherForecast files locally on 2026-10-28, but GitHub still showed them at `b5fc8f3` that day, so **push pending**. `Program.cs` still has `AddControllers()`/`MapControllers()`/`AddOpenApi()`. VS-generated `Dockerfile` (8080/8081), launch profiles (http 5071, https 7024; override locally with `--urls`).
  - `src/Frontend/outlay-web`: Vite 8 + React 19.3 + TypeScript ~6.0 + ESLint 10, no Tailwind yet. **Strict mode is already on**: TypeScript 6 defaults `strict` to `true`, verified with a TS 6.0.3 compile probe. An earlier note saying it was missing was wrong.
  - Thorough `.gitignore` (ignores `logs/`, `*.log`, `*.bak`, `.env`, `*.pfx`) and `.dockerignore`. No tests, `global.json`, `Directory.*.props`, CI, compose or README.
- **No dependency on the `csharp-code-quality` skill.** The owner turned it off, and all conventions are now stated directly in CLAUDE.md, TRD and ADRs.

### Not done / not verified
- The referenced YouTube short couldn't be viewed (no retrievable page content). The 6 core docs follow the common "spec before code" sequence and the exact file names the owner gave.
- No application code, tests or pipelines exist beyond the template.
- The template uses MVC controllers. The specs use Minimal API endpoints per slice (ADR-0004), so the controllers are removed in M0-01. If the owner prefers controllers, supersede ADR-0004 first.

---

## 2. Decisions (see ADRs)

| Topic | Decision |
|---|---|
| Architecture | Clean Architecture + Vertical Slices + CQRS/MediatR (14.x, Community licence), with a Minimal API endpoint per slice (ADR-0002/3/4) |
| Auth | OpenIddict 7 + ASP.NET Core Identity. Password + custom OTP grant for first-party clients. Rolling refresh tokens with reuse detection. Refresh token in an HttpOnly cookie for the SPA. (ADR-0005) |
| Passwords | BCrypt (enhanced, work factor 12), hashed not encrypted (ADR-0006) |
| Permissions | Role templates + subset-only delegated grants + cascade revocation + data scopes (ADR-0007) |
| Tenancy | Shared DB, `tenant_id` + EF Core 10 named query filters + SaveChanges guard (ADR-0008) |
| Databases | 7 providers in tiers. MySQL via Oracle's provider (Pomelo has no EF Core 10 build). Access is Windows-only. MongoDB is limited. **InMemory needs exactly one API instance.** (ADR-0009) |
| Cache | FusionCache (L1 + Redis/Memcached L2 + backplane, fail-safe) (ADR-0010) |
| Logs | Serilog sub-loggers + `Sinks.Map` → `logs/<category>/dd-MM-yyyy.txt`, diagnostics middleware, ExceptionAdvisor, query interceptor with caller tags (ADR-0011) |
| Offline | Dexie per user+tenant, outbox with idempotency keys, delta pull by `change_version` (ADR-0012) |
| Assistant | Deterministic intent/template engine with guardrails, reusing authorised queries. Web Speech API on the client. (ADR-0013) |
| Budget semantics | Yearly record = monthly allowance. **Accepted**: owner had no preference, recommended option kept. (ADR-0014) |
| Import | Validate-all-then-commit, annotated error workbook (ADR-0015) |
| Mapping | Manual `ToDto()` + projection expressions. No AutoMapper (licence + CVE-2026-32933). Mapperly as escape hatch. (ADR-0016) |
| **Tests** | **Default test stack: xUnit (.NET test projects) + NSubstitute + AwesomeAssertions.** One `Outlay.UnitTests` project, plus Integration (Testcontainers), Architecture (NetArchTest, no cross-slice refs), Performance (BenchmarkDotNet), Load and Stress (k6 with Redis/DB failure injection) (ADR-0017) |
| Frontend | React 19 + Vite + Tailwind v4 + TanStack + Zustand + Dexie + i18next. "Kantha Ledger" design. (ADR-0018) |

---

## 3. Open Questions

From `MASTER-SPECIFICATION.md §6`. None block M0.
- **Resolved:** Q-01 (monthly allowance), Q-02 (edit only Saved/Pending in an open month, then adjustments), Q-03 ("Over by ৳…"), Q-04 (Managers see the whole organisation by default, and an Admin can narrow to team), Q-06 (MediatR 14 + Community licence; owner's company qualifies), Q-13 (repo layout).
- Still open but with working assumptions: Q-05 (production SMS provider), Q-07…Q-12, Q-14.

Limitations and deliberate deviations from the brief (free SMS, Access, MongoDB, red log files, Firefox voice, iOS install, InMemory single instance, bcrypt wording, and others) are listed in `MASTER-SPECIFICATION.md §7`.

---

## 4. How to Continue

1. Copy the spec files into the repo (`CLAUDE.md`, `MASTER-*`, `ROADMAP.md`, `RELEASENOTES.md`, `AI-HANDOVER.md` at the root; `docs/` as is) and commit them.
2. Paste `MASTER-PROMPT.md §1`, then the **M0** milestone prompt (§2).
3. Finish **M0-01** (remove template code), then M0-02 onward. No files need moving, since the existing layout is kept.
4. Register the MediatR Community licence key and store it in secrets as `MediatR:LicenseKey` before M0-06.

---

## 5. Verification Hints for the Next Session

- After M0: `docker compose -f deploy/docker-compose.yml --profile postgres up --scale api=2`, then hit `/health/ready`, open `/scalar` (themed), and call a test endpoint that throws. A new file `logs/application-exceptions-logs/<today dd-MM-yyyy>.txt` should show Class/File/Method/Line and advice.
- `dotnet test tests/Backend/Outlay.UnitTests` and `tests/Backend/Outlay.ArchitectureTests` pass, using xUnit only (no NUnit/Moq/FluentAssertions packages in `Directory.Packages.props`).
- After M1: Postman `00 Auth & MFA` folder passes with OTP `111111`.

---

## 6. Risks to Watch

- Seven DB providers multiply effort. Keep Tier 3 (Access, MongoDB) strictly smoke-tested until GA.
- EF Core provider version alignment (MySQL, Access) can block framework upgrades. Keep providers isolated behind MSBuild switches.
- Offline sync conflict UX needs early user testing (M9).
- MediatR Community licence eligibility: re-check if the company passes US$5M revenue or US$10M outside capital.

---

## 7. Ideas Parking Lot (not in scope; needs a requirement ID before work)

- Receipt photo attachments with OCR.
- Web push for approval requests.
- Tenant-level custom fields on expenses.
- BFF (server-side token session) as an alternative to in-memory SPA tokens. It would need a new ADR and a server layer.

---

## 8. Session Log

| Date | Session | Summary |
|---|---|---|
| 2026-10-27 | Spec authoring | Full spec set (13 documents + 18 ADRs). Repo and video not accessible at first, so assumptions were recorded as Q-13/Q-14. |
| 2026-10-27 | Repo reconciliation | Read `mtabassum/Outlay` @ `b5fc8f3`. Specs moved to the real layout (`Outlay.slnx` at root, `src/Backend`, `src/Frontend/outlay-web`). Q-13 resolved. |
| 2026-10-27 | Guides + C4 | Added backend and frontend developer guides and C4-DIAGRAMS. ADR-0016 revised (AutoMapper licence + CVE). |
| 2026-10-27 | Spec revision | csharp-code-quality skill removed as a source. Test stack decided (ADR-0017 rewritten: xUnit + NSubstitute + AwesomeAssertions, k6 with failure injection, single `Outlay.UnitTests`). Q-01 resolved (ADR-0014 accepted). C4 gained the project/dependency view, log-pipeline dynamic view, InMemory single-instance rule, and LB routes and middleware detail. Limitations and deviations consolidated in MASTER-SPECIFICATION §7. Fixed Mermaid code fences that would have broken on GitHub, and every block now passes a strict check. |
| 2026-10-28 | Decisions | Q-02, Q-03, Q-04 resolved with the working assumptions. Q-06 resolved: MediatR 14 + Community licence. Corrected the TS strict note (TS 6 defaults strict on). Recorded the owner's local WeatherForecast deletion (push pending). Added suggested `.gitignore` migrations comment (Backend guide §6.1). |
| 2026-10-28 | Edit rules | Added expense edit/delete rules (BR-EXP-08…13, AppFlow F-03b, TRD §5.4 with `UpdateExpense`/`DeleteExpense`, `Expense.Revise` status table, 16 tests incl. ৳1,200 → ৳12,000). Approver scope via `IApproverResolver` (BR-NOT-04/05): team-only Managers neither receive nor see out-of-scope requests. Q-02/Q-04/Q-06 wording finalised. Plan M2-04, M4-01/02, M9-03 updated. Gave owner the replacement `.gitignore` migrations comment. |
| 2026-10-28 | Setup portability | "xUnit everywhere" reworded to "Default test stack (ADR-0017)". Original clone command and repo references kept (owner's decision). Default ports kept in docs with source + override notes. Ports made configurable: `--urls`/`ASPNETCORE_URLS` for the API, `VITE_API_PROXY_TARGET`/`VITE_DEV_PORT` via `.env.development.local`, compose host-port variables in `deploy/.env`, relative `VITE_API_BASE_URL`. CLAUDE.md §7a added. TRD connection strings left as is (owner's request). |

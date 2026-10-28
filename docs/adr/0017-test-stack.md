# ADR-0017: Test stack: xUnit, NSubstitute, AwesomeAssertions, Testcontainers, NetArchTest, BenchmarkDotNet, k6

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The brief requires unit, integration, performance, load and stress tests as **separate projects**, without naming tools. An earlier draft split unit testing across NUnit (domain) and xUnit + Moq (application). The owner then chose a single framework, using this layout:

| Project | Purpose |
|---|---|
| `Outlay.UnitTests` | Domain rules (soft/hard limit, 80% once, plan limits), validators, advisor rules, redactor, filter compiler |
| `Outlay.IntegrationTests` | End-to-end API per provider, tenant isolation, auth/lockout/MFA/token rotation, idempotency, import all-or-nothing, log files written |
| `Outlay.PerformanceTests` | Filter compiler, redactor, cache path, summary query |
| `Outlay.LoadTests` | Target RPS and p95 on read/write mixes |
| `Outlay.StressTests` | Spike, soak, Redis/DB failure injection (graceful degradation) |
| `Outlay.ArchitectureTests` | Layer rules, no cross-slice references |

That layout named FluentAssertions and "NBomber or k6". Licensing facts as of October 2026:
- **FluentAssertions 8+** is commercial (Xceed partnership, January 2025). 7.x stays Apache-2.0 but gets no new features.
- **AwesomeAssertions** is the Apache-2.0 community fork of FluentAssertions 7. It has the same `.Should()` API and is actively released.
- **NBomber** needs a paid licence for organisational use. **k6** is a free, open-source CLI tool.
- **Moq** had the 2023 SponsorLink incident (hashed developer emails collected at build time, later removed). **NSubstitute** is BSD-licensed.

## Decision

| Concern | Tool | Licence |
|---|---|---|
| Test framework, **every** .NET test project | **xUnit** (`[Fact]`, `[Theory]` + `[InlineData]` / `[MemberData]`) | Apache-2.0 |
| Test doubles | **NSubstitute** | BSD-3 |
| Assertions | **AwesomeAssertions** (`result.Should().Be(...)`) | Apache-2.0 |
| Real databases and Redis | **Testcontainers** + **Respawn** (reset between tests) | MIT / Apache-2.0 |
| HTTP pipeline | `WebApplicationFactory<Program>` | MIT |
| Architecture rules | **NetArchTest.Rules** | MIT |
| Micro-benchmarks | **BenchmarkDotNet** | MIT |
| Load and stress (incl. failure injection) | **k6** scripts + compose commands to stop/start Redis and the DB | AGPL-3.0 (used as a tool, not linked) |

Project layout (`tests/Backend/`): `Outlay.UnitTests` (folders `Domain/`, `Application/`, `Infrastructure/` mirroring `src/Backend`), `Outlay.IntegrationTests`, `Outlay.ArchitectureTests`, `Outlay.PerformanceTests`, `Outlay.LoadTests`, `Outlay.StressTests`, plus the `Outlay.Testing` helper library (builders, `FakeClock`, personas, Testcontainers fixtures).

Rules:
- To use a different library, update this ADR first. FluentAssertions 8+ and NBomber need paid licences for commercial use, so check licensing before proposing them.
- Domain tests use no doubles. Application tests substitute only `Application/Abstractions` interfaces.
- Time is always a `FakeClock`. No `Thread.Sleep` and no wall-clock time in tests.

## Consequences

**Positive**
- One framework and one set of attributes and conventions across all tests (KISS). Fewer packages to upgrade.
- No commercial licences in the test toolchain.
- Integration tests and unit tests share fixtures and helpers naturally.

**Negative**
- One unit test project grows large. Folders mirror production namespaces to keep it navigable, and `dotnet test --filter "FullyQualifiedName~Outlay.UnitTests.Domain"` runs a subset.
- k6 scripts are JavaScript, not C#. They're small and isolated.

## Alternatives considered

| Option | Why not |
|---|---|
| NUnit for domain + xUnit for the rest | Two frameworks for no strong benefit. Replaced by this ADR at the owner's request. |
| FluentAssertions 8 | Paid licence for commercial use |
| FluentAssertions 7.x (pinned) | Free but frozen. AwesomeAssertions continues the same API. |
| Moq | Works, but owner preference and NSubstitute's simpler syntax |
| NBomber | Paid licence for organisational use |
| Shouldly | Fine alternative. AwesomeAssertions chosen to match the `.Should()` style in the owner's reference layout. |

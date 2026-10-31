# ADR-0011: Categorised Serilog file logging + diagnostics middleware

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner requires five log types at `logs/<type>/dd-MM-yyyy.txt`, toggleable via appsettings/.env, a custom middleware, caller location (class/file/method/line), exact reasons and possible fixes, query logs with thresholds and timings, and no sensitive data.

## Decision

- Serilog with one sub-logger per category, filtered by a `LogCategory` property, writing through `Serilog.Sinks.Map` keyed on a `LogDate` (`dd-MM-yyyy`) property to get the exact file names. Options are read via `IOptionsMonitor` (hot reload).
- `UseOutlayDiagnostics()` = CorrelationId + RequestLogging + GlobalException middlewares. Exceptions resolve the first `Outlay.*` stack frame (embedded PDBs) and consult `ExceptionAdvisor` for probable cause and fixes.
- EF `DbCommandInterceptor` + `TagWithCaller()` (CallerMemberName/FilePath/LineNumber), Dapper executor and Mongo command events feed the query log, with a slow threshold (ANSI red on console, `[SLOW QUERY ⚠]` in files, red in the S-22 viewer).
- Masking policy for sensitive property names and `[Sensitive]` DTO members. Money amounts are masked.
- Optional central sink (Seq/OTLP) for multi-replica deployments.

## Consequences

+ Meets every logging requirement with standard tooling.
- Per-replica files. Mitigated by instance subfolders or the central sink.
- Text files can't show colour. Documented, and solved in the viewer.

## Alternatives considered

Built-in `ILogger` providers only (no categorised files); NLog (fine, but Serilog has a richer sink ecosystem).

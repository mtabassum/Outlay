# ADR-0007: Custom logging middleware with five file categories

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Need configurable per-category files named dd-MM-yyyy.txt, caller class/file/method/line, exception reason + fixes, slow-query highlighting, audit with IP, and strict redaction.

## Decision
Build a small in-house pipeline: IOutlayLogger<T> (caller-info attributes) → bounded Channel → single writer → logs/<category>/dd-MM-yyyy.txt. Middleware, a MediatR behavior and EF/Dapper/Mongo interceptors feed it. Redaction by key and pattern. Config hot-reload.

## Consequences
+ Matches requirements exactly, no sink quirks. − We own the code (kept small and well tested). Can later forward to Serilog/OpenTelemetry through an adapter.

# ADR-0013: Bulk import is all-or-nothing with a complete error report

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Users need to fix many errors in one pass and avoid partial data.

## Decision
Parse → validate every row → detect in-file and DB duplicates → collect all errors → single-transaction insert only if there are zero errors. Runs in a background worker; provider-optimal bulk insert; the error CSV mirrors the upload.

## Consequences
+ Predictable, user-friendly. − Large files hold memory/transactions longer → caps (MaxRows, MaxFileMb) and streaming.

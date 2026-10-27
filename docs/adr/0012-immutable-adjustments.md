# ADR-0012: Append-only adjustments instead of edits

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Original requirement: corrections are separate entries; originals never modified; negatives never shown.

## Decision
ExpenseAdjustments table with Direction (Reduce/Increase) and positive Amount; reports use net; the UI shows magnitude with a direction label. Rejected hard-limit expenses are removed but an immutable ApprovalRequests snapshot remains (A-09).

## Consequences
+ Audit-friendly. − Queries must account for adjustments; solved by aggregates.

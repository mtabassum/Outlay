# ADR-0014: Yearly budget record = monthly allowance

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The requirement states "only one budget per category per year" and has a "Set Yearly Budget — 2026" screen, yet its Expense Categories table is headed "Monthly Budget", the Monthly Summary shows the same amounts as each month's budget, and the 80% alert fires "once per month per category".

## Decision

Interpret the yearly budget record's amount as the **monthly allowance for every month of that year**. Monitoring, alerts and reports are per calendar month. The UI explains this on the Budget Setup screen. Confirmed 2026-10-27: when asked to choose between monthly allowance and annual total (Q-01), the owner stated no preference, so the recommended monthly allowance stands.

## Consequences

+ Consistent with every number in the requirement's mocks and example report.
- If the owner intends an annual total, the `BudgetMonitor` and report formulas change (isolated in one domain service).

## Alternatives considered

Annual total with monthly pro-rata (contradicts the mock numbers); separate monthly budget rows (contradicts "one per year").

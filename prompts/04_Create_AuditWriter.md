# Prompt 04 — Implement AuditWriter (shared by all three layers)

Implement a reusable AuditWriter. It owns ALL physical audit-table writes; pipeline code
never inserts into audit tables directly.

## Methods
startRun / completeRun / failRun
startSource / completeSource / failSource / markSourcePartial
writeStageSummary, writeRuleResults(batch), writeMergeSummary,
writeReconciliation, writeLineage, writeErrors(batch)

## Rules
- Append-only: every call appends new rows. No updates, no overwrites of audit history.
- If audit.enabled is false, every method is a no-op.
- Explicit column selection in exact Hive table order before insertInto (or SQL INSERT
  with named columns if our Hive version supports it — evaluate and pick the safer one).
- Validate DataFrame schema against the Hive table schema before writing; fail loudly at
  startup, not silently at runtime.
- Audit-write failures: log with full context, then continue — and if a business exception
  is already in flight, re-throw THAT one. Never mask it. Never write an audit row about a
  failed audit write (no recursion).
- Buffer where it matters: rule results and errors are written in batches; stage-summary
  rows for one source file may be buffered and written once per file to limit 1-row Spark
  jobs and small HDFS files. Run/source lifecycle rows may write immediately.
- Truncate error_message and record_payload per AuditConfig; payload only when enabled,
  masked per config. Never store credentials/secrets.
- Safe timestamp creation (no reliance on session timezone surprises).

## Deliver
1. AuditWriter implementation.
2. Exact column-order mapping for each of the 8 tables.
3. Schema-validation helper.
4. A short usage example (start run → start source → stage → complete).

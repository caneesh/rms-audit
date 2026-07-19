# Prompt 20 — Tests

Add tests using the project's existing ScalaTest + Spark test setup. Use temp
directories / a test database; never require production Hive locations.

Cover:
- Model ↔ DDL column mapping and AuditWriter column ordering (a test that reads the DDL
  files and compares against the case classes, so drift fails the build).
- run_id / batch_id generation; append-only semantics (a status transition adds a row,
  never mutates).
- Kill switch: audit.enabled=false → zero audit writes, pipeline output identical.
- Raw: message reconciliation (input = parsed + rejected), staging count match and
  mismatch paths (mismatch must NOT commit), commit-failure path retains staging,
  PARTIAL file outcome, all 13 RawTarget mappings present.
- Rerun: completed file skipped, force-rerun processes with new attempt_number,
  PARTIAL retry skips already-COMPLETED targets.
- Error auditing: batching, buffer cap, truncation, payload OFF by default, original
  exception preserved when an audit write fails.
- Rule results: one-pass counting produces correct counts for multiple rules, threshold
  → WARNED vs FAILED, sample-key cap respected.
- Reconciliation: MATCHED / EXPLAINED / MISMATCHED classification, unexplained
  difference fails the run.
- Lineage: expected edges written for a small three-layer fixture.
- No changes to existing Raw DataFrame schemas or output logic (golden-output test on a
  small fixture file with auditing ON vs OFF — business output must be identical).

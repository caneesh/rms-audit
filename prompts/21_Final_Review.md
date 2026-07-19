# Prompt 21 — Final review

Review the complete audit implementation across all three layers. Verify each item and
report findings by severity.

Schema and models:
- 8 audit tables have DDL; Scala models match exactly; append-only (no updated_ts, no
  UPDATE anywhere); DDL deployment order documented.

Safety invariants:
- Raw schemas unchanged; business transformations unchanged in all layers.
- Curated/Gold changed ONLY by the two tracking columns.
- audit.enabled=false restores original behavior everywhere (verified by the golden test).
- Audit failures never mask business failures; no recursive audit writes.
- No PHI in audit tables by default; truncation and caps enforced.

Correctness:
- All 13 Raw targets audited via one reusable function; staging validates before every
  commit; nothing commits on mismatch.
- input = processed + rejected enforced per source; reconciliation identities implemented
  with named reasons; unexplained differences fail runs.
- Rerun protection: completed files skipped, PARTIAL retries skip completed targets,
  audit history never overwritten; the unrecoverable mid-commit case is documented in a
  runbook note, not hidden.
- Rule counting is one-pass; no per-rule count jobs; no redundant scans of Raw/Curated/
  Gold tables for audit purposes.
- Lineage: batch edges complete across GOLD→CURATED→RAW→file; row-level trace doc with
  worked query exists.

Compatibility:
- Everything compiles against the Scala/Spark/Hive versions from prompt 01; no
  unsupported APIs; queries in the ops pack run on our Hive version.

Return: findings by severity, required fixes, optional improvements, full list of new/
modified files, a deployment checklist (DDL first, then jars, config flags staged with
audit.enabled=false → smoke test → true), and a rollback checklist (flip kill switch;
audit tables can stay in place).

# Prompt 09 — Analyze the Curated pipeline (shell/beeline)

Analyze the Curated pipeline implementation. **Do not make any code changes.**

This pipeline runs as shell scripts calling `hivebeeline`. The analysis must understand
the shell orchestration, not just the HiveQL.

Identify and report:

## Shell orchestration
1. Entry point script(s) and how they're triggered (trigger files from Raw? cron? manual?).
2. The master script flow: which scripts call which, in what order.
3. How table iteration works (`$goldTablesFromParams` pattern or similar).
4. Where `.prm` parameter files live and what variables they define.
5. Where `.hql` files live (the `${edgenode_hqlPath}` structure).
6. Existing logging pattern (`fnLogMsg` or similar) — where logs are written.
7. Existing error handling — what happens when `hivebeeline` returns non-zero?
8. How dependencies from Raw are detected (trigger files? partition checks?).

## Data flow (per Curated table)
9. Which Raw tables feed each Curated table (the source mapping).
10. How incremental scope is determined — trigger file content? timestamp column? partition?
11. The CDC/merge mechanism: ALTER TABLE SET LOCATION → MERGE INTO? Something else?
12. What `--hivevar` parameters are passed to merge HQL files.

## Business logic
13. Every place records are dropped, filtered, deduplicated — list each with business reason
    if discernible. These become named reconciliation reasons.
14. The business rules and DQ checks in the HQL (name them, note which HQL file).
15. The load-bearing JOINs (highest risk of row explosion or silent drops).

## Schema and config
16. Whether Curated schemas can accept `_audit_run_id`, `_audit_batch_id` columns.
17. Any existing run/batch identifier today (even informal).
18. Hive version (confirm MERGE INTO is supported natively).

## Output
- Shell script inventory with call graph
- Per-table: source mapping, CDC mechanism, HQL file locations
- Drop/filter inventory with reasons
- Rule inventory
- Schema-change feasibility
- Risks and integration points for audit instrumentation
- Recommended insertion points for audit calls in shell scripts

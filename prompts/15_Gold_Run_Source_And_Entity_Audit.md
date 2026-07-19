# Prompt 15 — Instrument Gold: run, source, and entity audit

Add run/source/stage audit to the Gold pipeline using the same AuditWriter
(layer='GOLD'). No business-logic changes. Kill switch applies.

- startRun with pipeline name and spark_application_id.
- One source_control row-pair per Curated input table per run: source_type='TABLE',
  the processing window used, input_count from the persisted read (count once).
- One stage_summary row per Gold target: input/expected/actual counts. actual count
  follows the write mechanism from prompt 14 (partition-scoped count or
  _audit_batch_id-tagged count — Gold carries the tracking columns).
- Add _audit_run_id / _audit_batch_id to Gold writes (per prompt 14 feasibility; note
  any table that cannot take them).
- completeRun / failRun with totals.
- Same diff-with-justification output format as prompts 05/10.

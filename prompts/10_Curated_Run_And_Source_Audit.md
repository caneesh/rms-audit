# Prompt 10 — Instrument Curated: run and source audit

Add run-level and source-level audit to the Curated pipeline using the SAME AuditWriter
(layer='CURATED'). Do not change transformations or business logic. Kill switch applies.

At pipeline start: run_id, startRun (pipeline_name = the curated pipeline,
spark_application_id from SparkContext).

Per source Raw table read (one audit_source_control row-pair per source table per run):
- source_type='TABLE', source_name = raw table, batch_id per curated target run.
- watermark_start / watermark_end = the incremental window actually used (from prompt 09
  findings).
- input_count = rows read from the Raw source within the window — count the persisted
  source DataFrame once; never scan the Raw table twice.

Add lineage: for each Curated target written, call writeLineage with one row per
contributing Raw source batch (join audit_source_control on the Raw side by table +
window to find the source batch ids; if that lookup is ambiguous, record source_name +
window and leave source_batch_id null — do not guess).

Add the two tracking columns to Curated writes: _audit_run_id, _audit_batch_id
(schema change confirmed in prompt 09; if a table cannot take the columns, record that
table in the response and audit it at batch level only).

Per Curated target table: stage_summary row with input/expected/actual counts using the
same persist-once / count-once discipline as Raw. Validation of written counts follows
the apply mechanism found in prompt 09 (e.g., count the overwritten partition, or rows
tagged with this _audit_batch_id — now possible because Curated carries the column).

At end: completeRun / failRun with totals.

Show modified sections as diffs with one-line safety justifications.

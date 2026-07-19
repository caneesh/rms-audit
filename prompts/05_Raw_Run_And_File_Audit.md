# Prompt 05 — Instrument XMLToHive: run and file audit

Add run-level and file-level audit instrumentation to XMLToHive using AuditWriter.
Do NOT change XML parsing, transformations, DataFrame schemas, table names, partitioning,
or Raw output behavior. If audit.enabled=false, behavior must be byte-for-byte today's.

At application start:
- Generate run_id. Call startRun with layer='RAW', pipeline/process name, source system,
  and spark_application_id from the active SparkContext.

Per Sequence File:
- Generate batch_id. Capture file name, path, size, modification time, and checksum if
  cheaply available. Determine attempt_number and prior_batch_id from existing
  audit_source_control history for this file identity (lookup only — skip/rerun logic
  comes in prompt 08).
- Call startSource.
- Count messages read (input_count), parsed OK (processed_count), rejected
  (rejected_count) — use counters/accumulators in the existing read loop; do NOT add a
  second pass over the file.
- Reconciliation policy: if input_count != processed_count + rejected_count, write an
  error_detail row (error_type='RECONCILIATION') and fail the FILE (not the whole run);
  continue with remaining files.
- completeSource / failSource / markSourcePartial based on the file outcome
  (PARTIAL = some of the 13 stage writes completed, some failed — wired up in prompt 06).

At application end:
- completeRun or failRun with total/completed/failed source counts and message totals
  aggregated from the per-file results.

Show every modified section as a before/after diff and explain in one line each why it
cannot change business behavior.

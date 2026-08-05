# Prompt 02a — Create audit DDL (control tables)

Create Hive DDL for audit control tables. These track run and source lifecycle.

**Rules:**
- Append-only (no UPDATE, no DELETE) — status changes are new rows
- `layer` column on every table: 'RAW', 'CURATED', or 'GOLD'
- Use `created_ts TIMESTAMP` only (no updated_ts)
- STRING for IDs, TIMESTAMP for times, BIGINT for counts
- Partition high-volume tables by load_date

**Hive version:** [PASTE: version from 01a]

## Table 1: audit_run_control
Tracks pipeline run lifecycle. One row per status change.

Columns:
- layer, run_id, pipeline_name, process_name, source_system, spark_application_id
- status (STARTED, VALIDATING, COMPLETED, FAILED, PARTIAL)
- start_time, end_time, duration_ms
- total_sources, completed_sources, failed_sources (BIGINT)
- total_input_count, total_processed_count, total_rejected_count (BIGINT)
- error_message, created_ts

## Table 2: audit_source_control
Tracks each input source (file or table). One row per status change.

Columns:
- layer, run_id, batch_id
- source_type (FILE or TABLE), source_name, source_path
- source_file_size, source_file_checksum, file_modified_time
- watermark_start, watermark_end (for table sources)
- input_count, processed_count, rejected_count (BIGINT)
- expected_target_count, completed_target_count, failed_target_count (INT)
- attempt_number, prior_batch_id
- count_validation_status, status, start_time, end_time, duration_ms
- error_message, created_ts

Create both tables with comments explaining the append-only model.

# Prompt 02 — Create the unified Hive audit schema (all layers)

Create production-ready Hive DDL for ONE shared audit schema used by the Raw, Curated,
and Gold pipelines. Use the Hive/Spark versions found in prompt 01. Use the repository's
existing conventions for database naming, storage format, external-vs-managed tables,
and locations (use configurable placeholders for locations).

## Global rules
- Configurable audit database; CREATE DATABASE IF NOT EXISTS; CREATE TABLE IF NOT EXISTS.
- **Append-only event model.** Tables have `created_ts` only — NO `updated_ts`, no updates.
  Status transitions are new rows; consumers read the latest row per key. Put this in the
  table comments.
- Every table has a `layer` STRING column ('RAW' | 'CURATED' | 'GOLD').
- STRING for ids and statuses, TIMESTAMP for times, BIGINT for counts/sizes/durations.
- Table and column comments on everything. Practical partitioning (e.g., a load_date
  partition on high-volume tables like error_detail and rule_result); do not over-partition.
- Document the status vocabulary in comments:
  run/source: STARTED → VALIDATING → COMPLETED | FAILED | PARTIAL
  stage:      STARTED → COMPLETED | FAILED ; validation: MATCHED | MISMATCHED
- No unsupported Hive features for our version.

## Tables and required columns

### audit_run_control
layer, run_id, pipeline_name, process_name, source_system, spark_application_id,
status, start_time, end_time, duration_ms,
total_sources BIGINT, completed_sources BIGINT, failed_sources BIGINT,
total_input_count BIGINT, total_processed_count BIGINT, total_rejected_count BIGINT,
error_message, created_ts

### audit_source_control
(one input unit: a Sequence File for RAW; a source table + window for CURATED/GOLD)
layer, run_id, batch_id, source_type ('FILE'|'TABLE'), source_name, source_path,
source_file_size BIGINT, source_file_checksum STRING, file_modified_time TIMESTAMP,
watermark_start TIMESTAMP, watermark_end TIMESTAMP,
input_count BIGINT, processed_count BIGINT, rejected_count BIGINT,
expected_target_count INT, completed_target_count INT, failed_target_count INT,
attempt_number INT, prior_batch_id STRING,
count_validation_status, status, start_time, end_time, duration_ms,
error_message, created_ts

### audit_stage_summary
layer, run_id, batch_id, source_name, stage_name, target_table,
input_count BIGINT, expected_output_count BIGINT, actual_written_count BIGINT,
count_difference BIGINT, reject_count BIGINT,
count_validation_status, status, start_time, end_time, duration_ms,
error_message, created_ts

### audit_rule_result
(business rules + data quality + referential integrity in ONE table)
layer, run_id, batch_id, target_table, rule_type ('BUSINESS'|'DQ'|'RI'),
rule_name, rule_description, passed_count BIGINT, failed_count BIGINT,
threshold_pct DOUBLE, sample_failed_keys STRING (capped, delimited),
status ('PASSED'|'FAILED'|'WARNED'), duration_ms, created_ts

### audit_merge_summary
(CDC / merge / SCD activity)
layer, run_id, batch_id, target_table, operation_type ('CDC'|'MERGE'|'SCD1'|'SCD2'|'PIT'),
inserted_count, updated_count, deleted_count, noop_count, expired_count,
late_arriving_count, out_of_order_count, duplicate_event_count,
status, start_time, end_time, duration_ms, error_message, created_ts
(all counts BIGINT)

### audit_reconciliation
layer, run_id, entity_name, from_layer, to_layer,
source_count BIGINT, target_count BIGINT, rejected_count BIGINT, filtered_count BIGINT,
difference BIGINT, expected_difference_reason STRING, unexplained_difference BIGINT,
status ('MATCHED'|'EXPLAINED'|'MISMATCHED'), created_ts

### audit_lineage
(batch-level edges only — never row-level)
layer, run_id, batch_id, target_table,
source_layer, source_run_id, source_batch_id, source_name, created_ts

### audit_error_detail
layer, run_id, batch_id, source_name, stage_name, target_table,
message_id STRING, record_index BIGINT, error_type, error_code, error_message,
record_payload STRING (nullable — capture is config-controlled, truncated, masked),
error_timestamp, created_ts

## Deliver
1. One SQL file per table + one master file (database + all tables, in order).
2. Commented-out DROP statements as rollback examples only.
3. DESCRIBE FORMATTED validation queries.
4. A short note explaining partitioning and storage-format choices.

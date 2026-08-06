# Prompt 02b — Create audit DDL (detail tables)

Create Hive DDL for the remaining 6 audit tables.

**Same rules as 02a:** Append-only, layer column, STRING for IDs, BIGINT for counts.

**Hive version:** [PASTE: version from 01a]

## Table 3: audit_stage_summary
Per-target write summary.

Columns:
- layer, run_id, batch_id, source_name, stage_name, target_table
- input_count, expected_output_count, actual_written_count, count_difference, reject_count (BIGINT)
- count_validation_status (MATCHED, MISMATCHED), status, start_time, end_time, duration_ms
- error_message, created_ts

## Table 4: audit_rule_result
Business rules, DQ checks, RI checks.

Columns:
- layer, run_id, batch_id, target_table
- rule_type (BUSINESS, DQ, RI), rule_name, rule_description
- passed_count, failed_count (BIGINT), threshold_pct (DOUBLE)
- sample_failed_keys STRING (comma-separated, capped at 20)
- status (PASSED, FAILED, WARNED), duration_ms, created_ts

Partition by load_date (high volume).

## Table 5: audit_merge_summary
CDC/merge/SCD statistics.

Columns:
- layer, run_id, batch_id, target_table
- operation_type (CDC, MERGE, SCD1, SCD2, PIT)
- inserted_count, updated_count, deleted_count, noop_count, expired_count (BIGINT)
- late_arriving_count, out_of_order_count, duplicate_event_count (BIGINT)
- status, start_time, end_time, duration_ms, error_message, created_ts

## Table 6: audit_reconciliation
Layer-to-layer count reconciliation.

Columns:
- layer, run_id, entity_name, from_layer, to_layer
- source_count, target_count, rejected_count, filtered_count, difference (BIGINT)
- expected_difference_reason STRING, unexplained_difference BIGINT
- status (MATCHED, EXPLAINED, MISMATCHED), created_ts

## Table 7: audit_lineage
Batch-level lineage edges (not row-level).

Columns:
- layer, run_id, batch_id, target_table
- source_layer, source_run_id, source_batch_id, source_name
- created_ts

## Table 8: audit_error_detail
Individual error records.

Columns:
- layer, run_id, batch_id, source_name, stage_name, target_table
- message_id, record_index (BIGINT)
- error_type, error_code, error_message
- record_payload STRING (nullable, config-controlled, truncated)
- error_timestamp, created_ts

Partition by load_date (high volume).

Create all 6 tables with comments.

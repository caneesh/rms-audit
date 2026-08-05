# Prompt 19a — Create operational queries

Create a pack of operational queries for monitoring and troubleshooting the audit data.

## Query 1: Recent run status
```sql
-- Last N runs per layer
SELECT 
  layer, run_id, pipeline_name, status,
  start_time, end_time, duration_ms / 1000 AS duration_sec,
  total_sources, completed_sources, failed_sources,
  error_message
FROM ${audit_db}.audit_run_control
WHERE created_ts >= DATE_SUB(current_date(), 7)
  AND status IN ('COMPLETED', 'FAILED')  -- terminal states only
ORDER BY created_ts DESC
LIMIT 50;
```

## Query 2: Failed sources detail
```sql
-- Sources that failed in the last N days
SELECT 
  layer, run_id, batch_id, source_name,
  input_count, processed_count, rejected_count,
  status, error_message, created_ts
FROM ${audit_db}.audit_source_control
WHERE status = 'FAILED'
  AND created_ts >= DATE_SUB(current_date(), 7)
ORDER BY created_ts DESC;
```

## Query 3: Rule failures summary
```sql
-- Rules that failed (grouped by rule)
SELECT 
  layer, target_table, rule_type, rule_name,
  COUNT(*) AS failure_count,
  SUM(failed_count) AS total_failed_records,
  AVG(failed_count) AS avg_failed_per_run
FROM ${audit_db}.audit_rule_result
WHERE status = 'FAILED'
  AND created_ts >= DATE_SUB(current_date(), 7)
GROUP BY layer, target_table, rule_type, rule_name
ORDER BY failure_count DESC;
```

## Query 4: Count mismatches
```sql
-- Stage summaries where counts didn't match
SELECT 
  layer, run_id, batch_id, target_table,
  expected_output_count, actual_written_count, count_difference,
  count_validation_status, created_ts
FROM ${audit_db}.audit_stage_summary
WHERE count_validation_status = 'MISMATCHED'
  AND created_ts >= DATE_SUB(current_date(), 7)
ORDER BY created_ts DESC;
```

## Query 5: Merge statistics trend
```sql
-- CDC/Merge activity over time
SELECT 
  layer, target_table, TO_DATE(created_ts) AS load_date,
  SUM(inserted_count) AS inserts,
  SUM(updated_count) AS updates,
  SUM(deleted_count) AS deletes,
  SUM(late_arriving_count) AS late_arriving
FROM ${audit_db}.audit_merge_summary
WHERE created_ts >= DATE_SUB(current_date(), 30)
GROUP BY layer, target_table, TO_DATE(created_ts)
ORDER BY load_date DESC, target_table;
```

## Query 6: Error detail drill-down
```sql
-- Errors for a specific run
SELECT 
  source_name, stage_name, target_table,
  error_type, error_code, error_message,
  message_id, record_index, error_timestamp
FROM ${audit_db}.audit_error_detail
WHERE run_id = '${run_id}'
ORDER BY error_timestamp;
```

## Query 7: Lineage trace
```sql
-- Trace a Gold batch back to source file
WITH gold_batch AS (
  SELECT batch_id FROM ${audit_db}.audit_source_control 
  WHERE layer = 'GOLD' AND run_id = '${run_id}' LIMIT 1
),
curated_batch AS (
  SELECT source_batch_id 
  FROM ${audit_db}.audit_lineage 
  WHERE layer = 'GOLD' AND batch_id = (SELECT batch_id FROM gold_batch)
),
raw_batch AS (
  SELECT source_batch_id 
  FROM ${audit_db}.audit_lineage 
  WHERE layer = 'CURATED' AND batch_id = (SELECT source_batch_id FROM curated_batch)
)
SELECT source_name, source_path, file_modified_time
FROM ${audit_db}.audit_source_control
WHERE layer = 'RAW' AND batch_id = (SELECT source_batch_id FROM raw_batch);
```

## Query 8: Rerun candidates
```sql
-- Files/sources that might need rerun (FAILED or PARTIAL)
SELECT 
  layer, run_id, batch_id, source_name,
  attempt_number, status, error_message, created_ts
FROM ${audit_db}.audit_source_control
WHERE status IN ('FAILED', 'PARTIAL')
  AND created_ts >= DATE_SUB(current_date(), 7)
ORDER BY created_ts DESC;
```

Create all 8 queries as a SQL file or query pack.

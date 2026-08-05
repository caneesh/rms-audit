# Prompt 12 — Curated CDC / merge audit (HiveQL with MERGE INTO)

Implement `audit_merge_summary` for the Curated apply step. This pipeline uses native
Hive MERGE INTO (confirmed in prompt 09).

## Capturing MERGE statistics

Hive MERGE INTO does not directly return row counts for INSERT/UPDATE/DELETE operations.
Two approaches:

### Option A: Classification before MERGE (preferred if already done)

If the CDC logic already classifies records as I/U/D before the MERGE (common pattern),
count from the classification:

```sql
-- Before MERGE: count by operation type from the staged CDC data
INSERT INTO ${audit_db}.audit_merge_summary
SELECT
  'CURATED' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  'MERGE' AS operation_type,
  SUM(CASE WHEN cdc_operation = 'I' THEN 1 ELSE 0 END) AS inserted_count,
  SUM(CASE WHEN cdc_operation = 'U' THEN 1 ELSE 0 END) AS updated_count,
  SUM(CASE WHEN cdc_operation = 'D' THEN 1 ELSE 0 END) AS deleted_count,
  SUM(CASE WHEN cdc_operation = 'N' THEN 1 ELSE 0 END) AS noop_count,  -- no change
  0 AS expired_count,
  SUM(CASE WHEN event_ts < '${watermark_start}' THEN 1 ELSE 0 END) AS late_arriving_count,
  -- out_of_order: event older than latest applied version for same key
  -- duplicate_event: same key+version seen multiple times
  -- (these require more complex logic, may be NULL in v1)
  NULL AS out_of_order_count,
  NULL AS duplicate_event_count,
  'COMPLETED' AS status,
  CAST('${start_time}' AS TIMESTAMP) AS start_time,
  current_timestamp() AS end_time,
  NULL AS duration_ms,
  NULL AS error_message,
  current_timestamp() AS created_ts
FROM ${cdc_staging_table}
WHERE batch_id = '${batch_id}';
```

### Option B: Before/after counts (fallback)

If CDC operation type is not available, count before and after MERGE:

```bash
# Before MERGE
before_count=$(audit_get_count "SELECT COUNT(*) FROM ${target_table} WHERE ${partition_condition}")

# Execute MERGE
hivebeeline -f ${merge_hql} --hivevar ...
merge_result=$?

# After MERGE
after_count=$(audit_get_count "SELECT COUNT(*) FROM ${target_table} WHERE ${partition_condition}")

# Calculate delta (imprecise: can't distinguish insert vs update)
net_change=$((after_count - before_count))
```

In this case, record what IS knowable:
- `inserted_count = NULL` (cannot distinguish)
- `updated_count = NULL`
- `deleted_count = NULL`
- Add a note: `error_message = 'Counts from before/after delta only'`
- Record `net_change` in a custom field or error_message

Be honest about what cannot be captured. Do NOT fabricate I/U/D breakdown.

## Late-arriving and out-of-order detection

If the data includes event timestamps:

```sql
-- Late arriving: event_ts older than the current watermark window
late_arriving_count = SUM(CASE WHEN event_ts < '${watermark_start}' THEN 1 ELSE 0 END)

-- Out of order: requires comparing against the latest applied version per key
-- This may require a self-join or window function over the CDC data
out_of_order_count = (
  SELECT COUNT(*) 
  FROM ${cdc_staging} c1
  WHERE EXISTS (
    SELECT 1 FROM ${target_table} t 
    WHERE t.business_key = c1.business_key 
      AND t.version_ts > c1.event_ts
  )
)

-- Duplicate events: same business_key + event_ts appearing multiple times
duplicate_event_count = (
  SELECT SUM(cnt - 1)
  FROM (
    SELECT business_key, event_ts, COUNT(*) as cnt
    FROM ${cdc_staging}
    GROUP BY business_key, event_ts
    HAVING COUNT(*) > 1
  ) dups
)
```

These can be expensive. If not affordable, record NULL and document the limitation.

## Shell integration

```bash
# Capture start time
merge_start_time=$(date -u +"%Y-%m-%d %H:%M:%S")

# Execute CDC classification counting (Option A) or before-count (Option B)
# ...

# Execute MERGE
fnLogMsg INFO "Executing MERGE for ${target_table}"
hivebeeline -f ${merge_hql} --hivevar hiveDB_cdc=${hiveDB_cdc} --hivevar hiveDB_current=${hiveDB_current} ...
merge_result=$?

merge_end_time=$(date -u +"%Y-%m-%d %H:%M:%S")

if [ ${merge_result} -ne 0 ]; then
  audit_write_merge_summary "CURATED" "${RUN_ID}" "${BATCH_ID}" "${target_table}" \
    "MERGE" "" "" "" "" "" "" "" "" \
    "FAILED" "${merge_start_time}" "${merge_end_time}" "" "MERGE failed with exit code ${merge_result}"
  audit_write_error "CURATED" "${RUN_ID}" "${BATCH_ID}" "" "MERGE" "${target_table}" \
    "" "" "MERGE_FAILURE" "${merge_result}" "MERGE INTO failed" "" 
  continue  # skip to next table
fi

# Write successful merge summary (counts from Option A or B)
audit_write_merge_summary "CURATED" "${RUN_ID}" "${BATCH_ID}" "${target_table}" \
  "MERGE" "${inserted_count}" "${updated_count}" "${deleted_count}" "${noop_count}" \
  "0" "${late_arriving_count}" "${out_of_order_count}" "${duplicate_event_count}" \
  "COMPLETED" "${merge_start_time}" "${merge_end_time}" "" ""
```

## Deliverables

1. HQL template for merge statistics capture (Option A: from CDC classification)
2. Shell logic for before/after counting (Option B: fallback)
3. Late-arriving and duplicate detection queries
4. Shell integration showing merge_summary write after each MERGE
5. Error handling for MERGE failures
6. Honest documentation of what cannot be captured

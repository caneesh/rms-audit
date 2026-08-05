# Prompt 15 — Instrument Gold shell scripts: run, source, and entity audit

Add run/source/stage audit to the Gold pipeline shell scripts using the same audit
library (prompt 03b). Layer='GOLD'. Kill switch applies.

Do not change HiveQL transformations or business logic.

## At pipeline start

```bash
source ${AUDIT_LIB_PATH}/audit_functions.sh
audit_init || { fnLogMsg ERROR "Audit init failed"; }

RUN_ID=$(audit_generate_run_id)
audit_start_run "GOLD" "${RUN_ID}" "${PIPELINE_NAME}" "${PROCESS_NAME}" "${SOURCE_SYSTEM}"
```

## Per Curated source table

Before processing each Curated input:
```bash
BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${curated_table}")

# Determine the processing window (from trigger file or parameters)
watermark_start="${CURATED_WATERMARK_START}"
watermark_end="${CURATED_WATERMARK_END}"

# Count input rows from Curated
input_count=$(audit_get_count "SELECT COUNT(*) FROM ${curated_table} WHERE ${watermark_condition}")

audit_start_source "GOLD" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${curated_table}" \
  "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
```

## Per Gold target table

After each Gold write (SCD apply, MERGE, etc.):
```bash
# Count written rows
# Option 1: If Gold has _audit_batch_id column
actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${gold_table} WHERE _audit_batch_id='${BATCH_ID}'")

# Option 2: If counting by partition
actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${gold_table} WHERE load_date='${current_date}'")

audit_write_stage_summary "GOLD" "${RUN_ID}" "${BATCH_ID}" "${curated_table}" \
  "SCD2" "${gold_table}" "${input_count}" "${expected_count}" "${actual_count}" \
  "$((expected_count - actual_count))" "0" \
  "$([ ${expected_count} -eq ${actual_count} ] && echo 'MATCHED' || echo 'MISMATCHED')" \
  "COMPLETED" "${stage_start}" "${stage_end}" "" ""
```

## Schema change: add audit columns to Gold tables

If prompt 14 confirmed schema changes are allowed:

1. Modify Gold DDL to add:
   - `_audit_run_id STRING`
   - `_audit_batch_id STRING`

2. Modify the SCD/MERGE HQL to populate them:
```sql
-- In INSERT or MERGE ... THEN INSERT
INSERT INTO ${gold_table}
SELECT 
  ...,
  '${run_id}' AS _audit_run_id,
  '${batch_id}' AS _audit_batch_id
FROM ...
```

Pass via `--hivevar run_id=${RUN_ID} --hivevar batch_id=${BATCH_ID}`.

For SCD2 version closure (UPDATE to set expiry date), the existing row's audit columns
remain unchanged — they record when that version was created.

Note any table that cannot accept the columns.

## At pipeline end

```bash
total_sources=$(audit_get_count "SELECT COUNT(DISTINCT source_name) FROM ${AUDIT_DB}.audit_source_control WHERE layer='GOLD' AND run_id='${RUN_ID}'")
completed_sources=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_source_control WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='COMPLETED'")
failed_sources=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_source_control WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='FAILED'")

if [ ${failed_sources} -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "${failed_sources}" "Some sources failed"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "0"
fi
```

## Deliverables

1. Modified Gold master script with audit calls at run boundaries
2. Modified table-loop with source/stage audit calls
3. HQL modifications to populate `_audit_run_id`, `_audit_batch_id`
4. Show modified sections as diffs with one-line safety justifications
5. Note any tables that cannot accept audit columns

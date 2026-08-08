# Prompt 10b — Add Curated source audit

Add source-level audit inside the table processing loop.

**Context:**
- Run-level audit is set up (from 10a)
- Each table processed is a "source" in audit terms
- We need to track: input counts, watermarks, success/failure

**Find the table loop and add:**

```bash
# Inside the table iteration loop (for each table in $tablesFromParams)
for source_table in ${tablesFromParams}; do
  
  TOTAL_SOURCES=$((TOTAL_SOURCES + 1))
  
  # Generate batch ID for this table
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${source_table}")
  
  # Get watermark values from trigger file or parameters
  # [Adjust based on your actual mechanism from prompt 09a]
  watermark_start="${WATERMARK_START}"
  watermark_end="${WATERMARK_END}"
  
  # Count input rows BEFORE processing
  input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_raw}.${source_table} WHERE ${watermark_condition}")
  
  # Start source audit
  audit_start_source "CURATED" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${source_table}" \
    "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
  
  stage_start=$(date -u +"%Y-%m-%d %H:%M:%S")
  
  # ========== EXISTING PROCESSING CODE ==========
  # Source parameter file
  source ${table_paramFile}
  
  # Execute HQL (your existing beeline calls)
  hivebeeline -f ${merge_hql} --hivevar hiveDB_cdc=${hiveDB_cdc} \
    --hivevar hiveDB_current=${hiveDB_current} \
    --hivevar run_id=${RUN_ID} \
    --hivevar batch_id=${BATCH_ID}
  merge_result=$?
  # ========== END EXISTING CODE ==========
  
  stage_end=$(date -u +"%Y-%m-%d %H:%M:%S")
  
  if [ ${merge_result} -ne 0 ]; then
    fnLogMsg ERROR "MERGE failed for ${source_table}"
    audit_fail_source "${RUN_ID}" "${BATCH_ID}" "${source_table}" "MERGE failed with exit code ${merge_result}"
    audit_write_error "CURATED" "${RUN_ID}" "${BATCH_ID}" "${source_table}" "MERGE" "" \
      "" "" "MERGE_FAILURE" "${merge_result}" "MERGE HQL failed" ""
    FAILED_SOURCES=$((FAILED_SOURCES + 1))
    continue  # or exit based on your error policy
  fi
  
  # Count output (if Curated has _audit_batch_id column)
  actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_current}.${target_table} WHERE _audit_batch_id='${BATCH_ID}'")
  
  # Write stage summary
  audit_write_stage_summary "CURATED" "${RUN_ID}" "${BATCH_ID}" "${source_table}" \
    "MERGE" "${target_table}" "${input_count}" "${input_count}" "${actual_count}" \
    "$((input_count - actual_count))" "0" \
    "$([ ${input_count} -eq ${actual_count} ] && echo 'MATCHED' || echo 'EXPLAINED')" \
    "COMPLETED" "${stage_start}" "${stage_end}" ""
  
  # Complete source
  audit_complete_source "${RUN_ID}" "${BATCH_ID}" "${source_table}" "${input_count}" "${actual_count}" "0" "1" "0"
  COMPLETED_SOURCES=$((COMPLETED_SOURCES + 1))
  
done
```

Show the diff of changes to your table processing loop.

[PASTE: Your table loop code here]

# Prompt 15a — Add Gold run/source audit

Add audit instrumentation to Gold scripts. Same pattern as Curated (10a, 10b).

**Context:**
- Shell audit functions exist (S02/S03)
- Gold uses similar patterns to Curated but with SCD logic
- layer = "GOLD"

**Add to Gold master script (same pattern as Curated 10a):**

```bash
#!/bin/bash
# Source audit
source /path/to/audit_config.sh
source /path/to/audit_functions.sh
audit_init

RUN_ID=$(audit_generate_run_id)
export RUN_ID

audit_start_run "GOLD" "${RUN_ID}" "GoldPipeline" "SCD_Load" "RMS"

TOTAL_SOURCES=0
COMPLETED_SOURCES=0
FAILED_SOURCES=0
```

**Add to table loop (source audit):**

```bash
for target_table in ${goldTablesFromParams}; do
  TOTAL_SOURCES=$((TOTAL_SOURCES + 1))
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${target_table}")
  
  # Input from Curated
  curated_source="${CURATED_TABLE_NAME}"
  input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${curated_source} WHERE ${watermark_condition}")
  
  audit_start_source "GOLD" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${curated_source}" \
    "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
  
  stage_start=$(date -u +"%Y-%m-%d %H:%M:%S")
  
  # ========== EXISTING SCD/MERGE PROCESSING ==========
  # Your existing Gold processing code
  # ====================================================
  
  stage_end=$(date -u +"%Y-%m-%d %H:%M:%S")
  
  # On success:
  actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_gold}.${target_table} WHERE _audit_batch_id='${BATCH_ID}'")
  
  audit_write_stage_summary "GOLD" "${RUN_ID}" "${BATCH_ID}" "${curated_source}" \
    "SCD2" "${target_table}" "${input_count}" "${input_count}" "${actual_count}" \
    "$((input_count - actual_count))" "0" "MATCHED" "COMPLETED" \
    "${stage_start}" "${stage_end}" ""
  
  audit_complete_source "${RUN_ID}" "${BATCH_ID}" "${input_count}" "${actual_count}" "0" "1" "0"
  COMPLETED_SOURCES=$((COMPLETED_SOURCES + 1))
done

# End of run
if [ ${FAILED_SOURCES} -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "${FAILED_SOURCES}" "Some tables failed"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "0"
fi
```

Show the diff of changes to your Gold scripts.

[PASTE: Your Gold master script or table loop]

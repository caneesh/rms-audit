# Prompt 13a — Add Curated reconciliation

Add reconciliation check at the end of Curated processing.

**Context:**
- Run and source audit are in place (from 10a, 10b)
- Rule results are written (from 11a)
- Reconciliation validates the accounting identity:
  `source_count = target_count + rejected_count + filtered_count`

**Add at the END of each table's processing (before moving to next table):**

```bash
# ============ RECONCILIATION ============
# All counts come from audit tables - NO new table scans

# Source count (already captured in source_control)
source_count=$(audit_get_count "
  SELECT COALESCE(SUM(input_count), 0) 
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
")

# Target count (from stage_summary)
target_count=$(audit_get_count "
  SELECT COALESCE(actual_written_count, 0) 
  FROM ${AUDIT_DB}.audit_stage_summary 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' AND target_table='${target_table}'
  ORDER BY created_ts DESC LIMIT 1
")

# Rejected count (from failed blocking rules)
rejected_count=$(audit_get_count "
  SELECT COALESCE(SUM(failed_count), 0) 
  FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND status='FAILED' 
    AND rule_name IN ('NULL_MID','NULL_CORP')  -- blocking rules
")

# Filtered count (from intentional business filters)
# Build the expected_difference_reason string
filtered_query="
  SELECT rule_name, failed_count 
  FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND rule_type='BUSINESS' 
    AND rule_name IN ('INACTIVE_MEMBER','FUTURE_EFFECTIVE')
"
filtered_count=0
expected_reason=""
while IFS=$'\t' read -r rule_name count; do
  [ -z "$count" ] && continue
  filtered_count=$((filtered_count + count))
  expected_reason="${expected_reason}${rule_name}:${count};"
done < <(${AUDIT_BEELINE_CMD} --silent=true -e "${filtered_query}" 2>/dev/null | grep -v "^$")

# Calculate unexplained difference
unexplained=$((source_count - target_count - rejected_count - filtered_count))

# Determine status
if [ ${unexplained} -eq 0 ]; then
  if [ ${filtered_count} -eq 0 ]; then
    recon_status="MATCHED"
  else
    recon_status="EXPLAINED"
  fi
else
  recon_status="MISMATCHED"
fi

# Write reconciliation
audit_write_reconciliation "CURATED" "${RUN_ID}" "${entity_name}" \
  "RAW" "CURATED" \
  "${source_table}" "${target_table}" \
  "${source_count}" "${target_count}" "${rejected_count}" "${filtered_count}" \
  "$((source_count - target_count))" "${expected_reason}" "${unexplained}" \
  "${recon_status}"

# MISMATCHED fails the run
if [ "${recon_status}" = "MISMATCHED" ]; then
  fnLogMsg ERROR "Reconciliation MISMATCHED for ${entity_name}: unexplained=${unexplained}"
  FAILED_SOURCES=$((FAILED_SOURCES + 1))
fi
```

Show how to integrate this into your table loop.

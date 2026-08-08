# Prompt 13 — Raw → Curated reconciliation (shell/HiveQL)

Implement `audit_reconciliation` for each Curated entity.

The invariant is an accounting identity, not raw_count == curated_count:

```
source_count = target_count + rejected_count + filtered_count (with named reasons)
```

## Counts (all from existing audit data — NO new table scans)

All inputs to this identity must come from numbers already computed during the run:

```bash
# source_count: from audit_source_control for this batch
source_count=$(audit_get_count "
  SELECT SUM(input_count) 
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
")

# target_count: from audit_stage_summary for this batch
target_count=$(audit_get_count "
  SELECT actual_written_count 
  FROM ${AUDIT_DB}.audit_stage_summary 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' AND target_table='${target_table}'
")

# rejected_count: sum of failed_count from blocking rules that reject rows
rejected_count=$(audit_get_count "
  SELECT COALESCE(SUM(failed_count), 0) 
  FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND status='FAILED' AND rule_name IN (${REJECTING_RULES})
")

# filtered_count: from rule results for intentional filters (non-blocking)
# Each filter reason is a rule; collect counts and reasons
filtered_query="
  SELECT rule_name, failed_count 
  FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND rule_type='BUSINESS' AND rule_name IN (${FILTER_RULES})
"
# Parse into: 'INACTIVE_MEMBER:1200;DEDUP:45'
filtered_count=0
expected_difference_reason=""
while IFS=$'\t' read -r rule_name count; do
  filtered_count=$((filtered_count + count))
  expected_difference_reason="${expected_difference_reason}${rule_name}:${count};"
done < <(hivebeeline --silent=true -e "${filtered_query}" 2>/dev/null | grep -v "^$")
```

## Calculate unexplained difference

```bash
unexplained_difference=$((source_count - target_count - rejected_count - filtered_count))

if [ ${unexplained_difference} -eq 0 ]; then
  if [ ${filtered_count} -eq 0 ]; then
    recon_status="MATCHED"
  else
    recon_status="EXPLAINED"
  fi
else
  recon_status="MISMATCHED"
fi
```

## Write reconciliation row

```bash
audit_write_reconciliation "CURATED" "${RUN_ID}" "${entity_name}" \
  "RAW" "CURATED" \
  "${source_count}" "${target_count}" "${rejected_count}" "${filtered_count}" \
  "$((source_count - target_count))" "${expected_difference_reason}" "${unexplained_difference}" \
  "${recon_status}"
```

Or via HQL:
```sql
INSERT INTO ${audit_db}.audit_reconciliation
SELECT
  'CURATED' AS layer,
  '${run_id}' AS run_id,
  '${entity_name}' AS entity_name,
  'RAW' AS from_layer,
  'CURATED' AS to_layer,
  ${source_count} AS source_count,
  ${target_count} AS target_count,
  ${rejected_count} AS rejected_count,
  ${filtered_count} AS filtered_count,
  ${source_count} - ${target_count} AS difference,
  '${expected_difference_reason}' AS expected_difference_reason,
  ${unexplained_difference} AS unexplained_difference,
  '${recon_status}' AS status,
  current_timestamp() AS created_ts;
```

## MISMATCHED policy

MISMATCHED fails the run — this is the Curated completion gate for counts.

```bash
if [ "${recon_status}" = "MISMATCHED" ]; then
  fnLogMsg ERROR "Reconciliation MISMATCHED for ${entity_name}: unexplained=${unexplained_difference}"
  RECON_FAILED=true
fi
```

## Curated completion gate

At the end of the Curated run, check all completion criteria:

```bash
# Check all gates
sources_failed=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_source_control WHERE run_id='${RUN_ID}' AND status='FAILED'")
blocking_rules_failed=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result WHERE run_id='${RUN_ID}' AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})")
merges_failed=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_merge_summary WHERE run_id='${RUN_ID}' AND status='FAILED'")
recon_mismatched=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_reconciliation WHERE run_id='${RUN_ID}' AND status='MISMATCHED'")
fatal_errors=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_error_detail WHERE run_id='${RUN_ID}' AND error_type IN ('FATAL','MERGE_FAILURE')")

if [ ${sources_failed} -gt 0 ] || [ ${blocking_rules_failed} -gt 0 ] || [ ${merges_failed} -gt 0 ] || [ ${recon_mismatched} -gt 0 ] || [ ${fatal_errors} -gt 0 ]; then
  error_reasons=""
  [ ${sources_failed} -gt 0 ] && error_reasons="${error_reasons}sources_failed:${sources_failed};"
  [ ${blocking_rules_failed} -gt 0 ] && error_reasons="${error_reasons}blocking_rules:${blocking_rules_failed};"
  [ ${merges_failed} -gt 0 ] && error_reasons="${error_reasons}merges_failed:${merges_failed};"
  [ ${recon_mismatched} -gt 0 ] && error_reasons="${error_reasons}recon_mismatched:${recon_mismatched};"
  [ ${fatal_errors} -gt 0 ] && error_reasons="${error_reasons}fatal_errors:${fatal_errors};"
  
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "${sources_failed}" "${error_reasons}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "0"
fi
```

## Deliverables

1. Reconciliation calculation logic in shell
2. HQL template for reconciliation INSERT
3. Filter/rejection rule configuration (which rules reject vs filter)
4. Completion gate check logic
5. Integration into master script

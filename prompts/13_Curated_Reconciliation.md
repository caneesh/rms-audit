# Prompt 13 — Raw → Curated reconciliation (shell/HiveQL)

Implement `audit_reconciliation` for each Curated entity.

The invariant is an accounting identity, not raw_count == curated_count:

```
source_count = target_count + rejected_count + filtered_count (with named reasons)
```

**Raw→Curated is one-to-one at table level**, so unlike Gold this layer needs no source
map, no driver/enrich roles and no fan-in gate — one source table, one target table, one
reconciliation row per batch. Do **not** apply `docs/GOLD_FANOUT_DESIGN.md` here. The
`source_name` / `target_table` columns still get populated, because the table is shared
with Gold, where they are what make a row identifiable.

Confirm the one-to-one assumption in prompt 09 rather than trusting it. If any Curated
table turns out to read two or more Raw tables, it has Gold's fan-in problem and the
design note applies after all — say so instead of quietly summing the inputs.

## Counts (all from existing audit data — NO new table scans)

All inputs to this identity must come from numbers already computed during the run:

**Take the latest row, never an aggregate over rows.** The audit model is append-only:
`audit_start_source` writes `input_count` and `audit_complete_source` writes it again, so
`SUM(input_count)` for a batch returns **twice** the real source count and every
reconciliation silently reports a huge unexplained difference. The same applies to
`stage_summary` on a retry.

```bash
# source_count: latest source_control row for this batch's source
source_count=$(audit_get_count "
  SELECT input_count
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE layer='CURATED' AND run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
    AND source_name='${source_table}'
  ORDER BY created_ts DESC LIMIT 1
")

# target_count: latest stage_summary row for this batch's target
target_count=$(audit_get_count "
  SELECT actual_written_count 
  FROM ${AUDIT_DB}.audit_stage_summary 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' AND target_table='${target_table}'
  ORDER BY created_ts DESC LIMIT 1
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
    AND rule_type='BUSINESS' AND rule_name IN (${FILTERING_RULES})
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
  "${source_table}" "${target_table}" \
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
  '${source_table}' AS source_name,
  '${target_table}' AS target_table,
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

Filter every gate query on `layer='CURATED'`. Run ids are unique in practice, but the
audit tables are shared by all three layers and an unfiltered gate silently widens as soon
as anything else writes under the same id.

```bash
# Sources not COMPLETED — latest row per source, since the model is append-only and a
# source that went STARTED then FAILED then COMPLETED on retry must count once, as COMPLETED.
sources_failed=$(audit_get_count "
  SELECT COUNT(*) FROM (
    SELECT batch_id, source_name, status FROM (
      SELECT batch_id, source_name, status,
             ROW_NUMBER() OVER (PARTITION BY batch_id, source_name ORDER BY created_ts DESC) rn
      FROM ${AUDIT_DB}.audit_source_control
      WHERE layer='CURATED' AND run_id='${RUN_ID}') t
    WHERE rn=1 AND status <> 'COMPLETED') x")

blocking_rules_failed=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result WHERE layer='CURATED' AND run_id='${RUN_ID}' AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})")
merges_failed=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_merge_summary WHERE layer='CURATED' AND run_id='${RUN_ID}' AND status='FAILED'")
recon_mismatched=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_reconciliation WHERE layer='CURATED' AND run_id='${RUN_ID}' AND status='MISMATCHED'")
fatal_errors=$(audit_get_count "SELECT COUNT(*) FROM ${AUDIT_DB}.audit_error_detail WHERE layer='CURATED' AND run_id='${RUN_ID}' AND error_type IN ('FATAL','MERGE_FAILURE')")

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

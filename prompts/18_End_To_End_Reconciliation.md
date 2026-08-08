# Prompt 18 — Curated → Gold reconciliation and end-to-end view

## Curated → Gold reconciliation (audit_reconciliation)

Same accounting-identity approach as prompt 13, per entity, with two Gold-specific twists:

### Twist 1: Fan-out is expected

One Curated row may legitimately produce rows in multiple Gold tables, and SCD2 writes
version rows. The identity is NOT raw row count = gold row count.

Define the per-entity expected relationship explicitly:
- "Curated members in scope = Gold member natural keys touched this batch"
- "Curated coverage rows = Gold coverage current versions created or updated"

Record this in `expected_difference_reason`.

### Twist 2: All counts from existing audit data

All counts come from numbers already computed during the run (`audit_stage_summary`,
`audit_merge_summary`, `audit_rule_result`). NO new table scans.

```bash
# source_count: Curated rows in scope (from source_control)
source_count=$(audit_get_count "
  SELECT SUM(input_count) 
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND source_name='${curated_table}'
")

# target_count: Gold natural keys affected, NOT raw row count
# Option A: From merge_summary (inserted + updated = keys touched)
target_count=$(audit_get_count "
  SELECT COALESCE(SUM(inserted_count + updated_count), 0)
  FROM ${AUDIT_DB}.audit_merge_summary 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' AND target_table='${gold_table}'
")

# Option B: From stage_summary actual_written_count (if SCD creates 1 row per key)
# Use whichever matches the entity's expected relationship

# rejected_count: from blocking rule failures
rejected_count=$(audit_get_count "
  SELECT COALESCE(SUM(failed_count), 0) 
  FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND status='FAILED' AND rule_name IN (${REJECTING_RULES})
")

# filtered_count: intentional drops (dedup, inactive, etc.)
# Build expected_difference_reason string as in prompt 13
```

### Write reconciliation

```sql
INSERT INTO ${audit_db}.audit_reconciliation
SELECT
  'GOLD' AS layer,
  '${run_id}' AS run_id,
  '${entity_name}' AS entity_name,
  'CURATED' AS from_layer,
  'GOLD' AS to_layer,
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

### MISMATCHED policy

MISMATCHED (unexplained_difference != 0) fails the Gold run.

## Gold completion gate

Final step reads this run's audit rows and marks the run COMPLETED only when ALL criteria
are met:

```bash
# 1. All Curated inputs processed
sources_incomplete=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_source_control 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status NOT IN ('COMPLETED')
")

# 2. Blocking rules passed
blocking_rules_failed=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' 
    AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})
")

# 3. RI checks passed (or within threshold)
ri_checks_failed=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' 
    AND rule_type='RI' AND status='FAILED'
")

# 4. History/merge succeeded
merges_failed=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_merge_summary 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='FAILED'
")

# 5. Reconciliation not MISMATCHED
recon_mismatched=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_reconciliation 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='MISMATCHED'
")

# 6. Lineage written
lineage_missing=$(audit_get_count "
  SELECT COUNT(DISTINCT target_table) 
  FROM ${AUDIT_DB}.audit_stage_summary 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='COMPLETED'
    AND target_table NOT IN (
      SELECT DISTINCT target_table FROM ${AUDIT_DB}.audit_lineage 
      WHERE layer='GOLD' AND run_id='${RUN_ID}'
    )
")

# 7. No fatal errors
fatal_errors=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_error_detail 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND error_type IN ('FATAL','MERGE_FAILURE','SCD_FAILURE')
")

# Evaluate gate
if [ ${sources_incomplete} -gt 0 ] || [ ${blocking_rules_failed} -gt 0 ] || \
   [ ${ri_checks_failed} -gt 0 ] || [ ${merges_failed} -gt 0 ] || \
   [ ${recon_mismatched} -gt 0 ] || [ ${lineage_missing} -gt 0 ] || [ ${fatal_errors} -gt 0 ]; then
  
  error_reasons=""
  [ ${sources_incomplete} -gt 0 ] && error_reasons="${error_reasons}sources_incomplete:${sources_incomplete};"
  [ ${blocking_rules_failed} -gt 0 ] && error_reasons="${error_reasons}blocking_rules_failed:${blocking_rules_failed};"
  [ ${ri_checks_failed} -gt 0 ] && error_reasons="${error_reasons}ri_failed:${ri_checks_failed};"
  [ ${merges_failed} -gt 0 ] && error_reasons="${error_reasons}merges_failed:${merges_failed};"
  [ ${recon_mismatched} -gt 0 ] && error_reasons="${error_reasons}recon_mismatched:${recon_mismatched};"
  [ ${lineage_missing} -gt 0 ] && error_reasons="${error_reasons}lineage_missing:${lineage_missing};"
  [ ${fatal_errors} -gt 0 ] && error_reasons="${error_reasons}fatal_errors:${fatal_errors};"
  
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "${sources_incomplete}" "${error_reasons}"
  fnLogMsg ERROR "Gold run FAILED: ${error_reasons}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "0"
  fnLogMsg INFO "Gold run COMPLETED successfully"
fi
```

## End-to-end reconciliation view

Create a SQL view (or query in the ops pack) that joins the RAW→CURATED and CURATED→GOLD
reconciliation rows per entity per day into one line:

```sql
CREATE VIEW IF NOT EXISTS ${audit_db}.v_reconciliation_daily AS
SELECT
  COALESCE(rc.entity_name, rg.entity_name) AS entity_name,
  TO_DATE(COALESCE(rc.created_ts, rg.created_ts)) AS load_date,
  
  -- RAW totals (from RAW source_control)
  rs.total_input AS raw_count,
  rs.total_processed AS raw_processed,
  rs.total_rejected AS raw_rejected,
  
  -- RAW → CURATED reconciliation
  rc.source_count AS raw_to_curated_source,
  rc.target_count AS curated_count,
  rc.rejected_count AS curated_rejected,
  rc.filtered_count AS curated_filtered,
  rc.expected_difference_reason AS curated_filter_reasons,
  rc.unexplained_difference AS curated_unexplained,
  rc.status AS curated_recon_status,
  
  -- CURATED → GOLD reconciliation
  rg.source_count AS curated_to_gold_source,
  rg.target_count AS gold_count,
  rg.rejected_count AS gold_rejected,
  rg.filtered_count AS gold_filtered,
  rg.expected_difference_reason AS gold_filter_reasons,
  rg.unexplained_difference AS gold_unexplained,
  rg.status AS gold_recon_status,
  
  -- Overall status
  CASE
    WHEN rc.status = 'MISMATCHED' OR rg.status = 'MISMATCHED' THEN 'MISMATCHED'
    WHEN rc.status = 'EXPLAINED' OR rg.status = 'EXPLAINED' THEN 'EXPLAINED'
    ELSE 'MATCHED'
  END AS overall_status

FROM (
  -- RAW run summary per entity per day
  SELECT 
    source_name AS entity_name,
    TO_DATE(created_ts) AS load_date,
    SUM(input_count) AS total_input,
    SUM(processed_count) AS total_processed,
    SUM(rejected_count) AS total_rejected
  FROM ${audit_db}.audit_source_control
  WHERE layer = 'RAW' AND status = 'COMPLETED'
  GROUP BY source_name, TO_DATE(created_ts)
) rs

LEFT JOIN ${audit_db}.audit_reconciliation rc
  ON rs.entity_name = rc.entity_name 
  AND rs.load_date = TO_DATE(rc.created_ts)
  AND rc.from_layer = 'RAW' AND rc.to_layer = 'CURATED'

LEFT JOIN ${audit_db}.audit_reconciliation rg
  ON rc.entity_name = rg.entity_name
  AND TO_DATE(rc.created_ts) = TO_DATE(rg.created_ts)
  AND rg.from_layer = 'CURATED' AND rg.to_layer = 'GOLD';
```

### Support query (single row per entity per day)

This is the single row support looks at each morning per entity:

```sql
-- Morning health check query
SELECT 
  entity_name,
  load_date,
  raw_count,
  curated_count,
  gold_count,
  curated_filter_reasons,
  gold_filter_reasons,
  curated_unexplained,
  gold_unexplained,
  overall_status
FROM ${audit_db}.v_reconciliation_daily
WHERE load_date = DATE_SUB(current_date(), 1)  -- yesterday's run
ORDER BY 
  CASE overall_status 
    WHEN 'MISMATCHED' THEN 1 
    WHEN 'EXPLAINED' THEN 2 
    ELSE 3 
  END,
  entity_name;
```

## Deliverables

1. Curated → Gold reconciliation logic with fan-out handling
2. Gold completion gate shell logic
3. `v_reconciliation_daily` view DDL
4. Support health-check query
5. Integration into Gold master script

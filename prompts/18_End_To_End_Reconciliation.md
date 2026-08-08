# Prompt 18 — Curated → Gold reconciliation and end-to-end view

## Inputs — read these before writing anything

- `docs/GOLD_ANALYSIS.md` (prompt 14) — the **per-edge expected relationship sentences**
  (these become `expected_difference_reason`), the **natural key per entity** (counts here
  are distinct keys, not rows), and `${REJECTING_RULES}`.
- `${AUDIT_DB}.audit_gold_source_map` — drives the DRIVER-edge loop and the lineage gate.
- `docs/GOLD_FANOUT_DESIGN.md` §5.

The reconciliation identity is the one thing in this pack that **cannot be derived from
the code** — it is a business statement about what the Gold table is supposed to contain.
If prompt 14 did not produce a sentence for an edge, ask for it. Do not infer one from
observed counts: that makes the check tautological and it will never catch a regression.

## Curated → Gold reconciliation (audit_reconciliation)

Same accounting-identity approach as prompt 13, but at a different grain. Read
`docs/GOLD_FANOUT_DESIGN.md` §5 first. Three Gold-specific twists:

### Twist 1: Reconcile per EDGE, never per layer

Curated→Gold is many-to-many. "curated_member → GOLD" is not a checkable statement,
because the same source table legitimately lands in several Gold targets under several
different rules. So write **one `audit_reconciliation` row per (source table → target
table) edge**, using the `source_name` and `target_table` columns added in prompt 02:

| edge | expected relationship |
|---|---|
| curated_member → gold_member_dim | 1:1 on member key |
| curated_member → gold_member_coverage | only members with active coverage; filter reason recorded |
| curated_member → gold_member_addr_hist | SCD2; version rows expected to exceed keys |

Each edge carries its own `expected_difference_reason` (from the prompt 14 inventory) and
is evaluated alone. **Never sum across edges** — a Curated table's count does not equal the
total across the Gold targets it feeds.

Only **DRIVER** edges get a reconciliation row. ENRICH and LOOKUP edges add columns rather
than rows, so counting them is meaningless; their correctness is audited as RI rules in
prompt 16. Read the roles from `audit_gold_source_map`.

### Twist 2: Count distinct natural keys, not rows

```
distinct_source_keys_in_scope
    = distinct_target_keys_touched + rejected_keys + filtered_keys
```

Row counts break in both directions: SCD2 inflates the target side with version rows, and
a fan-in enrichment source with multiple rows per key inflates the source side. Distinct
natural keys are stable under both.

### Twist 3: All counts from existing audit data

All counts come from numbers already computed during the run (`audit_stage_summary`,
`audit_merge_summary`, `audit_rule_result`). NO new table scans.

```bash
# Loop over DRIVER edges only — one reconciliation row each
for gold_table in $(audit_gold_targets_in_dependency_order); do
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${gold_table}")
  curated_table=$(audit_gold_driver_for "${gold_table}")

# source_count: Curated keys in scope for THIS edge.
# Must be scoped to BATCH_ID: with fan-out, the same source_name appears under several
# Gold batches, so summing across the run multiplies the source count by the fan-out.
source_count=$(audit_get_count "
  SELECT input_count
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE layer='GOLD' AND run_id='${RUN_ID}'
    AND batch_id='${BATCH_ID}' AND source_name='${curated_table}'
  ORDER BY created_ts DESC LIMIT 1
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
# Build expected_difference_reason string as in prompt 13 — one per EDGE, taken from the
# prompt 14 per-edge relationship sentences
done
```

### Write reconciliation

One row per DRIVER edge. `source_name` and `target_table` are what make the row
identifiable under fan-in and fan-out — without them, three rows for `curated_member`
would be indistinguishable.

```sql
INSERT INTO ${audit_db}.audit_reconciliation
SELECT
  'GOLD' AS layer,
  '${run_id}' AS run_id,
  '${entity_name}' AS entity_name,
  'CURATED' AS from_layer,
  'GOLD' AS to_layer,
  '${curated_table}' AS source_name,
  '${gold_table}' AS target_table,
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

# 6. Lineage written — COMPLETE, not merely present.
# Under fan-in, "at least one lineage edge exists" passes when 1 of 5 sources was recorded.
# Compare the edge count per target against the declared source count in the map.
lineage_incomplete=$(audit_get_count "
  SELECT COUNT(*) FROM (
    SELECT l.target_table
    FROM (SELECT target_table, COUNT(*) AS c
          FROM ${AUDIT_DB}.audit_lineage
          WHERE layer='GOLD' AND run_id='${RUN_ID}'
          GROUP BY target_table) l
    JOIN (SELECT gold_table, COUNT(*) AS c
          FROM ${AUDIT_DB}.audit_gold_source_map
          WHERE is_active='Y'
          GROUP BY gold_table) m
      ON l.target_table = m.gold_table
    WHERE l.c <> m.c) x
")

# Targets that completed a stage but wrote no lineage at all
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
   [ ${recon_mismatched} -gt 0 ] || [ ${lineage_missing} -gt 0 ] || \
   [ ${lineage_incomplete} -gt 0 ] || [ ${fatal_errors} -gt 0 ]; then
  
  error_reasons=""
  [ ${sources_incomplete} -gt 0 ] && error_reasons="${error_reasons}sources_incomplete:${sources_incomplete};"
  [ ${blocking_rules_failed} -gt 0 ] && error_reasons="${error_reasons}blocking_rules_failed:${blocking_rules_failed};"
  [ ${ri_checks_failed} -gt 0 ] && error_reasons="${error_reasons}ri_failed:${ri_checks_failed};"
  [ ${merges_failed} -gt 0 ] && error_reasons="${error_reasons}merges_failed:${merges_failed};"
  [ ${recon_mismatched} -gt 0 ] && error_reasons="${error_reasons}recon_mismatched:${recon_mismatched};"
  [ ${lineage_missing} -gt 0 ] && error_reasons="${error_reasons}lineage_missing:${lineage_missing};"
  [ ${lineage_incomplete} -gt 0 ] && error_reasons="${error_reasons}lineage_incomplete:${lineage_incomplete};"
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
reconciliation rows into one line.

**Grain: one row per Curated→Gold edge per day, not one row per entity per day.** Because
of fan-out, a Curated entity feeding three Gold tables produces three lines — one per
target, each with its own status and filter reasons. That is intended: collapsing them
would hide which target is mismatched. The RAW→CURATED columns repeat across an entity's
lines, so do not sum them.

```sql
CREATE VIEW IF NOT EXISTS ${audit_db}.v_reconciliation_daily AS
SELECT
  COALESCE(rc.entity_name, rg.entity_name) AS entity_name,
  rg.target_table AS gold_table,
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

-- Chain the hops on the Curated table itself: the target of RAW→CURATED is the source of
-- CURATED→GOLD. Matching on entity_name alone would cross-join a Curated entity against
-- every Gold edge that shares its name.
LEFT JOIN ${audit_db}.audit_reconciliation rg
  ON rc.target_table = rg.source_name
  AND TO_DATE(rc.created_ts) = TO_DATE(rg.created_ts)
  AND rg.from_layer = 'CURATED' AND rg.to_layer = 'GOLD';
```

### Support query (one row per Curated→Gold edge per day)

This is what support looks at each morning. An entity with fan-out shows one line per Gold
target, so a single mismatched target surfaces on its own line instead of being averaged
away:

```sql
-- Morning health check query
SELECT 
  entity_name,
  gold_table,
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
  entity_name,
  gold_table;
```

## Deliverables

1. Per-edge Curated → Gold reconciliation, DRIVER edges only, counted in distinct natural
   keys — with the per-edge `expected_difference_reason` taken from the prompt 14 inventory
2. Gold completion gate shell logic, including the `lineage_incomplete` check against
   `audit_gold_source_map`
3. `v_reconciliation_daily` view DDL at edge grain
4. Support health-check query
5. Integration into Gold master script

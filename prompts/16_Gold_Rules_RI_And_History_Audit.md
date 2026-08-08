# Prompt 16 — Gold rules, referential integrity, and history audit (HiveQL)

## Inputs — read these before writing anything

- `docs/GOLD_ANALYSIS.md` (prompt 14) — the rule inventory, RI relationships and their real
  foreign keys, the SCD action column name, and `${BLOCKING_RULES}`.
- `${AUDIT_DB}.audit_gold_source_map` — **ENRICH and LOOKUP edges are audited here, as RI
  rules.** Only DRIVER edges get reconciliation (prompt 18), so every non-driver input to a
  Gold table needs a matching `RI_*` rule in this prompt or it goes unaudited entirely.

Every rule name, column and join key below is a **placeholder** taken from a worked
example — `coverage_end_dt`, `member_key`, `scd_action`, `gold_member`. Replace all of them
with what prompt 14 actually found. Do not emit HQL referencing a column you have not seen
in the real schema.

## Rule results (audit_rule_result, layer='GOLD')

Wire rules from prompt 14 into `audit_rule_result` using the same one-pass HiveQL pattern
as Curated (prompt 11).

### Rules to cover

- **BUSINESS**: current coverage, latest member, primary address, subscriber selection,
  active product, effective-date logic (from prompt 14 inventory).
- **DQ**: missing MID, duplicate member, expired coverage on active record, invalid
  product, missing subscriber, bad effective date.
- **RI** (referential integrity): member exists, coverage exists, provider exists,
  product exists.

### One-pass counting (same pattern as prompt 11)

```sql
INSERT INTO ${audit_db}.audit_rule_result
SELECT
  'GOLD' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  rule_type, rule_name, rule_description,
  passed_count, failed_count,
  ${threshold_pct} AS threshold_pct,
  NULL AS sample_failed_keys,
  CASE 
    WHEN failed_count = 0 THEN 'PASSED'
    WHEN (failed_count * 100.0 / NULLIF(passed_count + failed_count, 0)) <= ${threshold_pct} THEN 'WARNED'
    ELSE 'FAILED' 
  END AS status,
  NULL AS duration_ms,
  current_timestamp() AS created_ts
FROM (
  -- Business rules
  SELECT 'BUSINESS', 'CURRENT_COVERAGE', 'Coverage must be current',
         SUM(CASE WHEN coverage_end_dt >= current_date() OR coverage_end_dt IS NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN coverage_end_dt < current_date() AND coverage_end_dt IS NOT NULL THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT 'BUSINESS', 'VALID_EFFECTIVE_DATE', 'Effective date must not be in future',
         SUM(CASE WHEN effective_dt <= current_date() THEN 1 ELSE 0 END),
         SUM(CASE WHEN effective_dt > current_date() THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  -- DQ rules
  UNION ALL
  
  SELECT 'DQ', 'NULL_MEMBER_ID', 'Member ID is required',
         SUM(CASE WHEN member_id IS NOT NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN member_id IS NULL THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT 'DQ', 'DUP_MEMBER_KEY', 'Duplicate member business key',
         COUNT(*) - COUNT(DISTINCT member_key),
         (SELECT COUNT(*) FROM (SELECT member_key FROM ${staging_table} GROUP BY member_key HAVING COUNT(*) > 1) d)
  FROM ${staging_table}
  
  -- Add more rules from prompt 14 inventory...
  
) AS rule_counts;
```

### RI checks (referential integrity)

RI rules use LEFT JOIN to find orphans:

```sql
-- RI: Member must exist in gold_member
UNION ALL

SELECT 'RI', 'RI_MEMBER_EXISTS', 'Member must exist in gold_member',
       COUNT(s.member_id),
       SUM(CASE WHEN m.member_id IS NULL THEN 1 ELSE 0 END)
FROM ${staging_table} s
LEFT JOIN ${gold_db}.gold_member m ON s.member_id = m.member_id AND m.is_current = 'Y'

UNION ALL

-- RI: Coverage must exist
SELECT 'RI', 'RI_COVERAGE_EXISTS', 'Coverage must exist in gold_coverage',
       COUNT(s.coverage_id),
       SUM(CASE WHEN c.coverage_id IS NULL THEN 1 ELSE 0 END)
FROM ${staging_table} s
LEFT JOIN ${gold_db}.gold_coverage c ON s.coverage_id = c.coverage_id AND c.is_current = 'Y'

-- Add provider, product RI checks from prompt 14...
```

**Performance note**: RI checks require JOINs to reference tables. If these are large,
consider:
- Only checking RI on new/changed records (filtered by `_audit_batch_id`)
- Running RI as a separate post-MERGE step
- Sampling (check random N records, not all)

Document any performance trade-offs made.

## History / SCD / PIT audit (audit_merge_summary, layer='GOLD')

Per Gold target, capture SCD operation counts from the DataFrames the SCD logic computes.

### SCD2 pattern

If the SCD2 logic classifies records before applying:

```sql
INSERT INTO ${audit_db}.audit_merge_summary
SELECT
  'GOLD' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  'SCD2' AS operation_type,
  SUM(CASE WHEN scd_action = 'INSERT' THEN 1 ELSE 0 END) AS inserted_count,
  SUM(CASE WHEN scd_action = 'UPDATE' THEN 1 ELSE 0 END) AS updated_count,
  0 AS deleted_count,
  SUM(CASE WHEN scd_action = 'NOOP' THEN 1 ELSE 0 END) AS noop_count,
  SUM(CASE WHEN scd_action = 'EXPIRE' THEN 1 ELSE 0 END) AS expired_count,
  NULL AS late_arriving_count,
  NULL AS out_of_order_count,
  NULL AS duplicate_event_count,
  'COMPLETED' AS status,
  CAST('${start_time}' AS TIMESTAMP) AS start_time,
  current_timestamp() AS end_time,
  NULL AS duration_ms,
  NULL AS error_message,
  current_timestamp() AS created_ts
FROM ${scd_staging_table};
```

### SCD1 pattern

For SCD1 (overwrite):
```sql
-- SCD1: only updated_count is meaningful
'SCD1' AS operation_type,
0 AS inserted_count,
COUNT(*) AS updated_count,  -- all rows are updates
...
```

### PIT (Point-in-Time) tables

If PIT tables exist:
```sql
'PIT' AS operation_type,
-- Use start_time to record the snapshot timestamp
CAST('${snapshot_ts}' AS TIMESTAMP) AS start_time,
SUM(CASE WHEN pit_action = 'CREATE' THEN 1 ELSE 0 END) AS inserted_count,
SUM(CASE WHEN pit_action = 'UPDATE' THEN 1 ELSE 0 END) AS updated_count,
SUM(CASE WHEN pit_action = 'CLOSE' THEN 1 ELSE 0 END) AS expired_count,
...
```

### History corrections

If a correction modifies effective dates on existing versions (unusual, may indicate
data quality issue):

```bash
# Detect history corrections: existing versions with changed effective dates
correction_count=$(audit_get_count "
  SELECT COUNT(*) FROM ${gold_table} g
  JOIN ${staging_table} s ON g.member_id = s.member_id AND g.version_id = s.version_id
  WHERE g.effective_dt != s.effective_dt
")

if [ ${correction_count} -gt 0 ]; then
  audit_write_error "GOLD" "${RUN_ID}" "${BATCH_ID}" "" "SCD2" "${target_table}" \
    "" "" "HISTORY_CORRECTION" "" "${correction_count} existing versions had effective dates changed" ""
  fnLogMsg WARN "History correction detected: ${correction_count} versions modified"
fi
```

## Shell integration

```bash
# After staging data is ready, before SCD apply
hivebeeline -f ${AUDIT_HQL_PATH}/gold_rules_${table}.hql \
  --hivevar audit_db=${AUDIT_DB} \
  --hivevar run_id=${RUN_ID} \
  --hivevar batch_id=${BATCH_ID} \
  --hivevar target_table=${gold_table} \
  --hivevar staging_table=${staging_table} \
  --hivevar gold_db=${GOLD_DB} \
  --hivevar threshold_pct=0

# Check for blocking failures
blocking_failures=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})
")

if [ ${blocking_failures} -gt 0 ]; then
  fnLogMsg ERROR "Blocking rule failures for ${gold_table}"
  # Fail every source row open on this batch — a blocking failure abandons the whole
  # target build, and under fan-in the batch has one open row per Curated input.
  for curated_table in $(audit_gold_sources_for "${gold_table}"); do
    audit_fail_source "${RUN_ID}" "${BATCH_ID}" "${curated_table}" \
      "Blocking rule failures: ${blocking_failures}"
  done
  continue
fi

# Execute SCD apply
# ...

# Write merge_summary after SCD completes
```

## Deliverables

1. `gold_rules_${table}.hql` with one-pass rule + RI counting
2. SCD merge_summary capture for SCD1, SCD2, PIT patterns
3. History correction detection
4. Shell integration for rule checking before SCD apply
5. Blocking rule configuration

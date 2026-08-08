# G16 — Gold rules, referential integrity and history audit (HiveQL)

## Baseline

Supersedes `prompts/16_Gold_Rules_RI_And_History_Audit.md`. Same one-pass counting pattern as
`prompts/11`, with `layer='GOLD'`.

## Inputs — read these before writing anything

- `docs/GOLD_ANALYSIS.md` (G14) — the rule inventory, RI relationships and their real foreign
  keys, the SCD action column name, and `${BLOCKING_RULES}`.
- `${AUDIT_DB}.audit_gold_source_map` — **ENRICH and LOOKUP edges are audited here, as RI
  rules.** Only DRIVER edges get reconciliation in G18, so every non-driver input needs a
  matching `RI_*` rule in this prompt or it goes completely unaudited.

**Every rule name, column and join key below is a placeholder** from a worked example —
`coverage_end_dt`, `member_key`, `scd_action`, `gold_member`. Replace all of them with what
G14 actually found. Do not emit HQL referencing a column you have not seen in the real
schema; say NOT FOUND instead.

## Which sources need RI rules

```sql
SELECT gold_table, curated_table, join_key
FROM ${AUDIT_DB}.audit_gold_source_map
WHERE gold_table='${target_table}' AND is_active='Y' AND source_role <> 'DRIVER';
```

Write one `RI_*` rule per row returned. If a Gold table has three lookups and you emit two RI
rules, one join is silently unaudited.

## Rule results — one pass

**Create file:** `audit_rules_gold_${table}.hql`

```sql
-- Gold rule + RI counting for ${target_table}
-- Run AFTER staging data is ready, BEFORE the SCD apply

INSERT INTO ${audit_db}.audit_rule_result
SELECT
  'GOLD' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  rule_type, rule_name, rule_description,
  passed_count, failed_count,
  CAST(${threshold_pct} AS DOUBLE) AS threshold_pct,
  NULL AS sample_failed_keys,
  CASE
    WHEN failed_count = 0 THEN 'PASSED'
    WHEN (CAST(failed_count AS DOUBLE) * 100.0 / NULLIF(passed_count + failed_count, 0)) <= ${threshold_pct} THEN 'WARNED'
    ELSE 'FAILED'
  END AS status,
  NULL AS duration_ms,
  current_timestamp() AS created_ts
FROM (

  -- ===== BUSINESS rules (from the G14 inventory) =====
  SELECT 'BUSINESS' AS rule_type,
         'CURRENT_COVERAGE' AS rule_name,
         'Coverage must be current' AS rule_description,
         CAST(SUM(CASE WHEN coverage_end_dt >= current_date() OR coverage_end_dt IS NULL THEN 1 ELSE 0 END) AS BIGINT) AS passed_count,
         CAST(SUM(CASE WHEN coverage_end_dt <  current_date() AND coverage_end_dt IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT) AS failed_count
  FROM ${staging_table}

  UNION ALL

  SELECT 'BUSINESS', 'VALID_EFFECTIVE_DATE', 'Effective date must not be in future',
         CAST(SUM(CASE WHEN effective_dt <= current_date() THEN 1 ELSE 0 END) AS BIGINT),
         CAST(SUM(CASE WHEN effective_dt >  current_date() THEN 1 ELSE 0 END) AS BIGINT)
  FROM ${staging_table}

  UNION ALL

  -- ===== DQ rules =====
  SELECT 'DQ', 'NULL_MEMBER_ID', 'Member ID is required',
         CAST(SUM(CASE WHEN member_id IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT),
         CAST(SUM(CASE WHEN member_id IS NULL     THEN 1 ELSE 0 END) AS BIGINT)
  FROM ${staging_table}

  UNION ALL

  -- ===== RI rules — one per non-DRIVER edge in audit_gold_source_map =====
  -- LEFT JOIN finds orphans.
  SELECT 'RI', 'RI_MEMBER_EXISTS', 'Member must exist in gold_member',
         CAST(SUM(CASE WHEN m.member_id IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT),
         CAST(SUM(CASE WHEN m.member_id IS NULL     THEN 1 ELSE 0 END) AS BIGINT)
  FROM ${staging_table} s
  LEFT JOIN ${gold_db}.gold_member m
    ON s.member_id = m.member_id AND m.is_current = 'Y'

  -- Add one RI block per remaining non-DRIVER edge...

) AS rule_counts;
```

### RI performance

RI checks join reference tables that may be large. Bound them:

- check RI only on new/changed records, filtered by `_audit_batch_id`
- or run RI as a separate post-merge step
- or sample

Document whichever trade-off you make. A rule HQL that runs longer than the load it audits
will be the first thing switched off.

## History / SCD audit (`audit_merge_summary`, layer='GOLD')

Capture what the SCD apply actually did. Only possible if the SCD logic records a per-row
action — G14 should have named that column.

```sql
INSERT INTO ${audit_db}.audit_merge_summary
SELECT
  'GOLD', '${run_id}', '${batch_id}', '${target_table}',
  'SCD2' AS operation_type,
  CAST(SUM(CASE WHEN scd_action = 'INSERT' THEN 1 ELSE 0 END) AS BIGINT) AS inserted_count,
  CAST(SUM(CASE WHEN scd_action = 'UPDATE' THEN 1 ELSE 0 END) AS BIGINT) AS updated_count,
  CAST(0 AS BIGINT) AS deleted_count,
  CAST(SUM(CASE WHEN scd_action = 'NOOP'   THEN 1 ELSE 0 END) AS BIGINT) AS noop_count,
  CAST(SUM(CASE WHEN scd_action = 'EXPIRE' THEN 1 ELSE 0 END) AS BIGINT) AS expired_count,
  NULL, NULL, NULL,
  'COMPLETED' AS status,
  CAST('${start_time}' AS TIMESTAMP), current_timestamp(), NULL, NULL,
  current_timestamp() AS created_ts
FROM ${scd_staging_table};
```

`inserted_count + updated_count` is what G18 uses as **distinct Gold keys touched** — the
target side of reconciliation. If the SCD logic cannot produce per-row actions, **say so
explicitly**: G18 then has to fall back to `stage_summary.actual_written_count`, which is only
valid when the SCD writes one row per key.

For **SCD1**, `operation_type='SCD1'` and all rows are updates. For **PIT** tables, use
`operation_type='PIT'` with the snapshot timestamp in `start_time`.

### History corrections

A correction that changes effective dates on existing versions is unusual and usually signals
a data quality problem. Detect and record it:

```bash
correction_count=$(audit_get_count "
  SELECT COUNT(*) FROM ${gold_table} g
  JOIN ${staging_table} s ON g.member_id = s.member_id AND g.version_id = s.version_id
  WHERE g.effective_dt != s.effective_dt")

if [ ${correction_count} -gt 0 ]; then
  audit_write_error "GOLD" "${RUN_ID}" "${BATCH_ID}" "" "SCD2" "${target_table}" \
    "" "" "HISTORY_CORRECTION" "" "${correction_count} existing versions had effective dates changed" ""
  fnLogMsg WARN "History correction detected: ${correction_count} versions modified"
fi
```

## Shell integration

```bash
hivebeeline -f ${AUDIT_HQL_PATH}/audit_rules_gold_${table}.hql \
  --hivevar audit_db=${AUDIT_DB} --hivevar run_id=${RUN_ID} \
  --hivevar batch_id=${BATCH_ID} --hivevar target_table=${gold_table} \
  --hivevar staging_table=${staging_table} --hivevar gold_db=${GOLD_DB} \
  --hivevar threshold_pct=0

blocking_failures=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
    AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})")

if [ ${blocking_failures} -gt 0 ]; then
  fnLogMsg ERROR "Blocking rule failures for ${gold_table}"
  # Fail every source row open on this batch — a blocking failure abandons the whole target
  # build, and under fan-in the batch has one open row per Curated input.
  for curated_table in $(audit_gold_sources_for "${gold_table}"); do
    audit_fail_source "${RUN_ID}" "${BATCH_ID}" "${curated_table}" \
      "Blocking rule failures: ${blocking_failures}"
  done
  continue
fi

# ... existing SCD apply ...
# ... then the merge_summary INSERT above ...
```

## Deliverables

1. `audit_rules_gold_${table}.hql` with one-pass rule + RI counting.
2. One RI rule per non-DRIVER edge, cross-checked against `audit_gold_source_map` — and a
   named list of any edge you could not write a rule for.
3. SCD merge_summary capture for the SCD1 / SCD2 / PIT patterns actually present.
4. History correction detection.
5. Shell integration, with blocking rules checked before the SCD apply.
6. Blocking rule configuration, and the RI performance trade-off you chose.

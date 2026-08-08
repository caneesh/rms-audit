# Prompt 16a — Create Gold rule + RI counting HQL, and SCD merge summary

Create HiveQL that counts all Gold rules in ONE table scan, plus the SCD activity capture.

**Context:**
- Rules, RI relationships and the SCD action column come from prompt 14a
  (`docs/GOLD_ANALYSIS.md`) — read it first
- Same UNION ALL single-scan pattern as 11a, with `layer='GOLD'`
- `${AUDIT_DB}.audit_gold_source_map` is loaded (from 14a's seed SQL)

**Every column name below is a placeholder from a worked example.** Replace them with what
14a actually found. Do not emit HQL referencing a column you have not seen in the real
schema — say NOT FOUND instead.

## Which sources get RI rules

This is the part that is easy to miss. Under fan-in a Gold table is built from several
Curated tables, but only the **DRIVER** gets a reconciliation identity in 18a. Every
**ENRICH** and **LOOKUP** edge is audited *here* or not at all:

```sql
SELECT gold_table, curated_table, join_key
FROM ${AUDIT_DB}.audit_gold_source_map
WHERE gold_table='${target_table}' AND is_active='Y' AND source_role <> 'DRIVER';
```

Write one `RI_*` rule per row returned. If a Gold table has three lookups and you emit two
RI rules, one join is silently unaudited.

**Create file:** `audit_rules_gold_${table}.hql`

```sql
-- Gold rule + RI counting for ${target_table}
-- Call AFTER staging data is ready, BEFORE the SCD apply
-- hivebeeline -f this_file.hql --hivevar audit_db=audit --hivevar run_id=... etc.

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

  -- ===== BUSINESS rules (from 14a inventory) =====
  SELECT 'BUSINESS' AS rule_type,
         'CURRENT_COVERAGE' AS rule_name,
         'Coverage must be current' AS rule_description,
         CAST(SUM(CASE WHEN coverage_end_dt >= current_date() OR coverage_end_dt IS NULL THEN 1 ELSE 0 END) AS BIGINT) AS passed_count,
         CAST(SUM(CASE WHEN coverage_end_dt <  current_date() AND coverage_end_dt IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT) AS failed_count
  FROM ${staging_table}

  UNION ALL

  -- ===== DQ rules =====
  SELECT 'DQ', 'NULL_MEMBER_ID', 'Member ID is required',
         CAST(SUM(CASE WHEN member_id IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT),
         CAST(SUM(CASE WHEN member_id IS NULL     THEN 1 ELSE 0 END) AS BIGINT)
  FROM ${staging_table}

  UNION ALL

  -- ===== RI rules — one per non-DRIVER edge in audit_gold_source_map =====
  -- LEFT JOIN finds orphans. Filter the staging side to this batch so the join stays
  -- bounded; a full-table RI check on a large Gold dimension is the usual cause of a
  -- rule HQL that runs longer than the load it audits.
  SELECT 'RI', 'RI_MEMBER_EXISTS', 'Member must exist in gold_member',
         CAST(SUM(CASE WHEN m.member_id IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT),
         CAST(SUM(CASE WHEN m.member_id IS NULL     THEN 1 ELSE 0 END) AS BIGINT)
  FROM ${staging_table} s
  LEFT JOIN ${gold_db}.gold_member m
    ON s.member_id = m.member_id AND m.is_current = 'Y'

  -- Add one RI block per remaining non-DRIVER edge...

) AS rule_counts;
```

## SCD merge summary

After the SCD apply completes, capture what it did. Only meaningful if the SCD logic
records a per-row action; 14a should have named that column.

```sql
-- audit_merge_summary_gold_${table}.hql
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

`inserted_count + updated_count` is what 18a uses as **distinct Gold keys touched** — the
target side of reconciliation. If your SCD logic cannot produce per-row actions, say so
explicitly: 18a then has to fall back to `stage_summary.actual_written_count`, which is
only valid when the SCD writes one row per key.

For **SCD1**, `operation_type='SCD1'` and all rows are updates. For **PIT** tables, use
`operation_type='PIT'` and put the snapshot timestamp in `start_time`.

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
  # Fail every source row on this batch — under fan-in there is one per Curated input
  for curated_table in $(audit_gold_sources_for "${gold_table}"); do
    audit_fail_source "${RUN_ID}" "${BATCH_ID}" "${curated_table}" \
      "Blocking rule failures: ${blocking_failures}"
  done
  continue
fi

# ... existing SCD apply ...
```

Create the rule HQL for one Gold table. List every rule and RI check you found, and name
any non-DRIVER edge you could not write an RI rule for.

[PASTE: Your Gold transformation / SCD HQL for one table]

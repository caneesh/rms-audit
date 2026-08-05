# Prompt 04b — HiveQL audit INSERT templates and counting patterns

Create reusable HiveQL templates for audit operations. These are used by the shell audit
library (prompt 03b) and can also be called directly from `.hql` files via `--hivevar`.

## INSERT templates (one per audit table)

Each template is a parameterized INSERT statement. Parameters are passed via `--hivevar`.

Example usage from shell:
```bash
hivebeeline -f ${AUDIT_HQL_PATH}/insert_run_control.hql \
  --hivevar audit_db=${AUDIT_DB} \
  --hivevar layer=CURATED \
  --hivevar run_id=${RUN_ID} \
  --hivevar pipeline_name=${PIPELINE_NAME} \
  ...
```

Or inline via the shell library:
```bash
hivebeeline -e "INSERT INTO ${AUDIT_DB}.audit_run_control SELECT ..."
```

### Templates to create:

1. `insert_run_control.hql` — run lifecycle (STARTED, COMPLETED, FAILED)
2. `insert_source_control.hql` — source/batch lifecycle
3. `insert_stage_summary.hql` — per-target write summary
4. `insert_rule_result.hql` — single rule result
5. `insert_rule_results_batch.hql` — multiple rules from a temp table or VALUES clause
6. `insert_merge_summary.hql` — CDC/merge statistics
7. `insert_reconciliation.hql` — layer-to-layer reconciliation
8. `insert_lineage.hql` — batch lineage edge
9. `insert_error_detail.hql` — single error
10. `insert_errors_batch.hql` — multiple errors from temp table

## One-pass rule counting pattern (HiveQL)

The Scala prompts require `.agg(sum(when(...)))` — here's the HiveQL equivalent.
This pattern counts ALL rules in a SINGLE table scan:

```sql
-- Count all rules for a target in one pass
INSERT INTO ${audit_db}.audit_rule_result
SELECT
  '${layer}',
  '${run_id}',
  '${batch_id}',
  '${target_table}',
  rule_type,
  rule_name,
  rule_description,
  passed_count,
  failed_count,
  ${threshold_pct},
  sample_keys,
  CASE WHEN failed_count = 0 THEN 'PASSED'
       WHEN failed_count <= total_count * ${threshold_pct} / 100 THEN 'WARNED'
       ELSE 'FAILED' END AS status,
  ${duration_ms},
  current_timestamp()
FROM (
  SELECT
    'DQ' AS rule_type,
    'NULL_MID' AS rule_name,
    'Member ID is null' AS rule_description,
    SUM(CASE WHEN mid IS NOT NULL THEN 1 ELSE 0 END) AS passed_count,
    SUM(CASE WHEN mid IS NULL THEN 1 ELSE 0 END) AS failed_count,
    COUNT(*) AS total_count,
    NULL AS sample_keys
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT
    'DQ' AS rule_type,
    'FUTURE_DOB' AS rule_name,
    'Date of birth is in the future' AS rule_description,
    SUM(CASE WHEN dob <= current_date() THEN 1 ELSE 0 END) AS passed_count,
    SUM(CASE WHEN dob > current_date() THEN 1 ELSE 0 END) AS failed_count,
    COUNT(*) AS total_count,
    NULL AS sample_keys
  FROM ${staging_table}
  
  -- Add more rules as UNION ALL...
) rules;
```

## Sample key collection pattern

Collecting sample failed keys requires a separate bounded query per rule (acceptable):
```sql
-- Collect up to N sample keys for a failed rule
SELECT CONCAT_WS(',', COLLECT_SET(mid))
FROM (
  SELECT mid
  FROM ${staging_table}
  WHERE mid IS NULL  -- rule condition
  LIMIT ${sample_cap}
) samples;
```

## Count capture pattern

For capturing counts into shell variables:
```sql
-- Return just the count (shell captures via grep)
SELECT COUNT(*) FROM ${table} WHERE ${condition};
```

Shell captures:
```bash
count=$(hivebeeline --silent=true -e "SELECT COUNT(*) FROM ..." 2>/dev/null | grep -E '^[0-9]+$')
```

## MERGE statistics capture

When using Hive MERGE INTO, capture statistics from the MERGE output or via before/after:
```sql
-- Before merge: capture target count
SET hivevar:before_count = (SELECT COUNT(*) FROM ${target_table} WHERE ${partition_condition});

-- After merge: calculate deltas
-- inserted = new rows (not in before)
-- updated = rows with changed values
-- This requires the MERGE to track operation type, OR compare before/after snapshots
```

If MERGE doesn't expose statistics directly, document what CAN be captured and what
must be NULL (don't fabricate counts).

## Deliverables

1. All 10 HiveQL template files
2. `rule_counting_template.hql` — the one-pass pattern with placeholder rules
3. `sample_keys_template.hql` — bounded sample collection
4. Documentation: which templates support batch vs single-row, parameter list for each

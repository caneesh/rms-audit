# Prompt S04 — Create HiveQL audit templates

Create HiveQL templates for one-pass rule counting. These are called from shell scripts via `hivebeeline -f`.

## Template 1: One-pass rule counting

**Create file:** `audit_rules_template.hql`

This counts ALL rules for a table in a SINGLE table scan using UNION ALL.

```sql
-- One-pass rule counting template
-- Call with: hivebeeline -f audit_rules.hql --hivevar audit_db=audit --hivevar run_id=... etc.

INSERT INTO ${audit_db}.audit_rule_result
SELECT
  '${layer}' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  rule_type,
  rule_name,
  rule_description,
  passed_count,
  failed_count,
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
  -- DQ Rule: Null check
  SELECT 'DQ' AS rule_type, 
         'NULL_MID' AS rule_name, 
         'Member ID is null' AS rule_description,
         SUM(CASE WHEN mid IS NOT NULL THEN 1 ELSE 0 END) AS passed_count,
         SUM(CASE WHEN mid IS NULL THEN 1 ELSE 0 END) AS failed_count
  FROM ${staging_table}
  
  UNION ALL
  
  -- DQ Rule: Another null check
  SELECT 'DQ', 'NULL_CORP', 'Corp code is null',
         SUM(CASE WHEN corp_cd IS NOT NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN corp_cd IS NULL THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  UNION ALL
  
  -- DQ Rule: Future date check
  SELECT 'DQ', 'FUTURE_DOB', 'Date of birth is in future',
         SUM(CASE WHEN dob <= current_date() OR dob IS NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN dob > current_date() THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  UNION ALL
  
  -- Business Rule: Active status check
  SELECT 'BUSINESS', 'INACTIVE_MEMBER', 'Member is inactive',
         SUM(CASE WHEN status = 'ACTIVE' THEN 1 ELSE 0 END),
         SUM(CASE WHEN status != 'ACTIVE' OR status IS NULL THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  -- Add more rules as UNION ALL blocks
  -- Each rule scans the same ${staging_table} but Hive optimizes into single scan
  
) AS rule_counts;
```

## Template 2: RI (Referential Integrity) checks

**Create file:** `audit_ri_checks.hql`

```sql
-- RI checks template
-- Checks that foreign keys exist in reference tables

INSERT INTO ${audit_db}.audit_rule_result
SELECT
  '${layer}', '${run_id}', '${batch_id}', '${target_table}',
  'RI' AS rule_type,
  rule_name,
  rule_description,
  passed_count,
  failed_count,
  ${threshold_pct},
  NULL,
  CASE WHEN failed_count = 0 THEN 'PASSED'
       WHEN (failed_count * 100.0 / NULLIF(passed_count + failed_count, 0)) <= ${threshold_pct} THEN 'WARNED'
       ELSE 'FAILED' END,
  NULL,
  current_timestamp()
FROM (
  -- RI: Member must exist
  SELECT 'RI_MEMBER_EXISTS' AS rule_name,
         'Member must exist in reference table' AS rule_description,
         SUM(CASE WHEN ref.member_id IS NOT NULL THEN 1 ELSE 0 END) AS passed_count,
         SUM(CASE WHEN ref.member_id IS NULL THEN 1 ELSE 0 END) AS failed_count
  FROM ${staging_table} stg
  LEFT JOIN ${ref_db}.member ref ON stg.member_id = ref.member_id
  
  UNION ALL
  
  -- RI: Coverage must exist
  SELECT 'RI_COVERAGE_EXISTS',
         'Coverage must exist in reference table',
         SUM(CASE WHEN ref.coverage_id IS NOT NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN ref.coverage_id IS NULL THEN 1 ELSE 0 END)
  FROM ${staging_table} stg
  LEFT JOIN ${ref_db}.coverage ref ON stg.coverage_id = ref.coverage_id
  
) AS ri_counts;
```

## Template 3: Sample failed keys collection

**Create file:** `audit_sample_keys.hql`

```sql
-- Collect sample failed keys for a specific rule
-- Returns comma-separated list of up to N keys

SELECT CONCAT_WS(',', COLLECT_LIST(key_value))
FROM (
  SELECT CAST(${key_column} AS STRING) AS key_value
  FROM ${staging_table}
  WHERE ${failure_condition}  -- e.g., "mid IS NULL"
  LIMIT ${sample_cap}
) samples;
```

Create all three template files. Adjust column names as placeholders that will be filled via --hivevar.

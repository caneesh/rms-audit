# Prompt 11a — Create Curated rule counting HQL

Create HiveQL to count all rules for a Curated table in ONE table scan.

**Context:**
- Rules identified in prompt 09b analysis
- Must use UNION ALL pattern for single-scan efficiency
- Called from shell via `hivebeeline -f`

**Rules to implement (adjust based on your 09b findings):**

| Rule Type | Rule Name | Condition |
|-----------|-----------|-----------|
| DQ | NULL_MID | mid IS NULL |
| DQ | NULL_CORP | corp_cd IS NULL |
| DQ | FUTURE_DOB | dob > current_date() |
| DQ | INVALID_GENDER | gender NOT IN ('M','F','U') |
| BUSINESS | INACTIVE_MEMBER | status != 'ACTIVE' |
| BUSINESS | FUTURE_EFFECTIVE | effective_dt > current_date() |
| DQ | DUP_BUSINESS_KEY | duplicate on (member_id, effective_dt) |

**Create file:** `audit_rules_curated_${table}.hql`

```sql
-- Curated rule counting for ${target_table}
-- Call AFTER staging data is ready, BEFORE final MERGE
-- hivebeeline -f this_file.hql --hivevar audit_db=audit --hivevar run_id=... etc.

INSERT INTO ${audit_db}.audit_rule_result
SELECT
  'CURATED' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  rule_type,
  rule_name,
  rule_description,
  passed_count,
  failed_count,
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
  
  -- [Add your rules here as UNION ALL blocks]
  -- Example:
  SELECT 'DQ' AS rule_type, 
         'NULL_MID' AS rule_name, 
         'Member ID is required' AS rule_description,
         CAST(SUM(CASE WHEN mid IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT) AS passed_count,
         CAST(SUM(CASE WHEN mid IS NULL THEN 1 ELSE 0 END) AS BIGINT) AS failed_count
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT 'DQ', 'NULL_CORP', 'Corp code is required',
         CAST(SUM(CASE WHEN corp_cd IS NOT NULL THEN 1 ELSE 0 END) AS BIGINT),
         CAST(SUM(CASE WHEN corp_cd IS NULL THEN 1 ELSE 0 END) AS BIGINT)
  FROM ${staging_table}
  
  -- Add more rules from your 09b analysis...
  
) AS rule_counts;
```

Create rule HQL for one Curated table. List all the rules you found in analysis.

[PASTE: Your Curated transformation HQL where you can see what validations exist]

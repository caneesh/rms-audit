# Prompt 11 — Curated business-rule and data-quality audit (HiveQL)

Wire the rules found in prompt 09 into `audit_rule_result`. Do NOT change what any rule
does — only count and record it.

## Rules to cover (from prompt 09 inventory)

- **BUSINESS**: inactive members removed, invalid products rejected, missing corp codes,
  future effective dates, duplicate coverage, bad enrollments, ...
- **DQ**: null MID, null corp, invalid DOB / future DOB, invalid gender, invalid state,
  missing coverage, ...
- **DEDUP**: duplicate business keys, duplicate source records, duplicate CDC events —
  record as DQ rules with rule_name prefixed 'DUP_'.

## The one-pass rule (mandatory)

Count ALL rules for a target in a SINGLE table scan using HiveQL aggregation.
Do NOT execute one query per rule.

Create a new HQL file (e.g., `audit_rules_${table}.hql`) or add to existing:

```sql
-- One-pass rule counting for ${target_table}
-- Called after staging data is ready, before final MERGE

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
  ${threshold_pct} AS threshold_pct,
  NULL AS sample_failed_keys,  -- collected separately if needed
  CASE 
    WHEN failed_count = 0 THEN 'PASSED'
    WHEN (failed_count * 100.0 / NULLIF(passed_count + failed_count, 0)) <= ${threshold_pct} THEN 'WARNED'
    ELSE 'FAILED' 
  END AS status,
  NULL AS duration_ms,
  current_timestamp() AS created_ts
FROM (
  SELECT 'DQ' AS rule_type, 'NULL_MID' AS rule_name, 'Member ID is null' AS rule_description,
         SUM(CASE WHEN mid IS NOT NULL THEN 1 ELSE 0 END) AS passed_count,
         SUM(CASE WHEN mid IS NULL THEN 1 ELSE 0 END) AS failed_count
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT 'DQ', 'NULL_CORP', 'Corp code is null',
         SUM(CASE WHEN corp_cd IS NOT NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN corp_cd IS NULL THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT 'DQ', 'FUTURE_DOB', 'Date of birth is in the future',
         SUM(CASE WHEN dob <= current_date() OR dob IS NULL THEN 1 ELSE 0 END),
         SUM(CASE WHEN dob > current_date() THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  UNION ALL
  
  SELECT 'BUSINESS', 'INACTIVE_MEMBER', 'Member is inactive',
         SUM(CASE WHEN status = 'ACTIVE' THEN 1 ELSE 0 END),
         SUM(CASE WHEN status != 'ACTIVE' THEN 1 ELSE 0 END)
  FROM ${staging_table}
  
  -- Add all rules from prompt 09 inventory as UNION ALL...
  
) AS rule_counts;
```

Execute from shell:
```bash
hivebeeline -f ${AUDIT_HQL_PATH}/audit_rules_${table}.hql \
  --hivevar audit_db=${AUDIT_DB} \
  --hivevar run_id=${RUN_ID} \
  --hivevar batch_id=${BATCH_ID} \
  --hivevar target_table=${target_table} \
  --hivevar staging_table=${staging_table} \
  --hivevar threshold_pct=0
```

## Sample failed keys (selective)

For rules with failures, collect sample keys in a separate bounded query:
```sql
-- Collect up to 20 sample keys for NULL_MID failures
SELECT CONCAT_WS(',', COLLECT_LIST(mid_key)) 
FROM (
  SELECT CAST(record_id AS STRING) AS mid_key
  FROM ${staging_table}
  WHERE mid IS NULL
  LIMIT 20
) samples;
```

Then UPDATE the audit_rule_result row (if Hive ACID is available) or write a correction row.
Alternatively, skip samples for simplicity in v1 — document the limitation.

## Join audit (selective)

Only for the load-bearing joins identified in prompt 09. Add to the same one-pass query:
```sql
UNION ALL

SELECT 'DQ', 'JOIN_MEMBER_COVERAGE', 'Member-coverage join validation',
       SUM(CASE WHEN coverage_id IS NOT NULL THEN 1 ELSE 0 END),  -- matched
       SUM(CASE WHEN coverage_id IS NULL THEN 1 ELSE 0 END)       -- unmatched (left-only)
FROM ${member_coverage_joined_staging}
```

## Rule outcome policy

- Each rule carries `threshold_pct` from config (default 0 = any failure fails).
- `failed_count` within threshold → status='WARNED'
- Above threshold → status='FAILED'
- A FAILED blocking rule fails the target's stage.

In shell, after rule HQL completes, check for blocking failures:
```bash
blocking_failures=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result 
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' 
    AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})
")
if [ ${blocking_failures} -gt 0 ]; then
  fnLogMsg ERROR "Blocking rule failures detected, skipping MERGE"
  audit_fail_source "${RUN_ID}" "${BATCH_ID}" "${source_table}" "Blocking rule failures"
  continue  # skip to next table
fi
```

Define blocking vs warning rules in config, not hardcoded.

## Deliverables

1. `audit_rules_${table}.hql` template with one-pass counting pattern
2. Shell integration showing where rule HQL is called in the table loop
3. Blocking-rule check logic
4. List of all rules to implement (from prompt 09 inventory)

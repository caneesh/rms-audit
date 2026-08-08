# Prompt 18b — Create end-to-end reconciliation view

Create a SQL view that shows the full RAW → CURATED → GOLD reconciliation across layers.

**This is what support looks at each morning.**

**Grain: one row per Curated→Gold edge per day**, not one row per entity per day. Because
of fan-out, an entity feeding three Gold tables produces three lines — one per target, each
with its own status and filter reasons. That is intended: collapsing them would hide which
target is mismatched. The RAW and CURATED columns repeat across an entity's lines, so do
not sum them.

```sql
-- End-to-end reconciliation view
-- One row per Curated→Gold edge per day showing counts across all layers

CREATE VIEW IF NOT EXISTS ${audit_db}.v_reconciliation_daily AS
SELECT
  COALESCE(raw_recon.entity_name, cur_recon.entity_name, gold_recon.entity_name) AS entity_name,
  gold_recon.target_table AS gold_table,
  COALESCE(raw_recon.load_date, cur_recon.load_date, gold_recon.load_date) AS load_date,
  
  -- RAW layer
  raw_recon.input_count AS raw_input,
  raw_recon.processed_count AS raw_processed,
  raw_recon.rejected_count AS raw_rejected,
  raw_recon.status AS raw_status,
  
  -- RAW → CURATED reconciliation
  cur_recon.source_count AS raw_to_curated_source,
  cur_recon.target_count AS curated_count,
  cur_recon.rejected_count AS curated_rejected,
  cur_recon.filtered_count AS curated_filtered,
  cur_recon.expected_difference_reason AS curated_filter_reasons,
  cur_recon.unexplained_difference AS curated_unexplained,
  cur_recon.status AS curated_recon_status,
  
  -- CURATED → GOLD reconciliation
  gold_recon.source_count AS curated_to_gold_source,
  gold_recon.target_count AS gold_count,
  gold_recon.rejected_count AS gold_rejected,
  gold_recon.filtered_count AS gold_filtered,
  gold_recon.expected_difference_reason AS gold_filter_reasons,
  gold_recon.unexplained_difference AS gold_unexplained,
  gold_recon.status AS gold_recon_status,
  
  -- Overall status
  CASE
    WHEN cur_recon.status = 'MISMATCHED' OR gold_recon.status = 'MISMATCHED' THEN 'MISMATCHED'
    WHEN cur_recon.status = 'EXPLAINED' OR gold_recon.status = 'EXPLAINED' THEN 'EXPLAINED'
    WHEN raw_recon.status = 'FAILED' THEN 'RAW_FAILED'
    ELSE 'MATCHED'
  END AS overall_status

FROM (
  -- RAW source summary per entity per day
  SELECT 
    source_name AS entity_name,
    TO_DATE(created_ts) AS load_date,
    SUM(input_count) AS input_count,
    SUM(processed_count) AS processed_count,
    SUM(rejected_count) AS rejected_count,
    MAX(status) AS status
  FROM ${audit_db}.audit_source_control
  WHERE layer = 'RAW'
  GROUP BY source_name, TO_DATE(created_ts)
) raw_recon

LEFT JOIN (
  -- RAW → CURATED reconciliation
  SELECT 
    entity_name, source_name, target_table,
    TO_DATE(created_ts) AS load_date,
    source_count, target_count, rejected_count, filtered_count,
    expected_difference_reason, unexplained_difference, status
  FROM ${audit_db}.audit_reconciliation
  WHERE from_layer = 'RAW' AND to_layer = 'CURATED'
) cur_recon 
  ON raw_recon.entity_name = cur_recon.entity_name 
  AND raw_recon.load_date = cur_recon.load_date

LEFT JOIN (
  -- CURATED → GOLD reconciliation, one row per edge
  SELECT 
    entity_name, source_name, target_table,
    TO_DATE(created_ts) AS load_date,
    source_count, target_count, rejected_count, filtered_count,
    expected_difference_reason, unexplained_difference, status
  FROM ${audit_db}.audit_reconciliation
  WHERE from_layer = 'CURATED' AND to_layer = 'GOLD'
) gold_recon
  -- Chain on the Curated TABLE: the target of RAW→CURATED is the source of CURATED→GOLD.
  -- Matching on entity_name alone would cross-join an entity against every Gold edge
  -- that happens to share its name.
  ON cur_recon.target_table = gold_recon.source_name
  AND cur_recon.load_date = gold_recon.load_date;
```

Also create a morning health check query:

```sql
-- Morning health check: yesterday's run status
SELECT 
  entity_name,
  gold_table,
  raw_input,
  curated_count,
  gold_count,
  curated_filter_reasons,
  gold_filter_reasons,
  curated_unexplained,
  gold_unexplained,
  overall_status
FROM ${audit_db}.v_reconciliation_daily
WHERE load_date = DATE_SUB(current_date(), 1)
ORDER BY 
  CASE overall_status 
    WHEN 'MISMATCHED' THEN 1
    WHEN 'RAW_FAILED' THEN 2
    WHEN 'EXPLAINED' THEN 3 
    ELSE 4 
  END,
  entity_name,
  gold_table;
```

An entity with fan-out shows one line per Gold target, so a single mismatched target
surfaces on its own line instead of being averaged away.

Create both the view DDL and the health check query.

# Prompt 17 — Gold lineage (batch-level + traceability keys)

Implement lineage so any Gold record can be traced to its MQ-origin file. Two parts:

## 1. Batch-level edges (audit_lineage)

For every Gold target written, record one row per contributing Curated source batch.

### Find Curated batches that fed this Gold batch

Query the Curated layer's `audit_source_control` to find batches within the processing window:

```bash
# Get Curated batches that overlap Gold's watermark window
curated_batches_query="
  SELECT run_id, batch_id, source_name 
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE layer='CURATED' 
    AND status='COMPLETED'
    AND source_name='${curated_table}'
    AND watermark_end >= '${gold_watermark_start}'
    AND watermark_start <= '${gold_watermark_end}'
"

# Write lineage for each contributing Curated batch
hivebeeline --silent=true -e "${curated_batches_query}" 2>/dev/null | while IFS=$'\t' read -r cur_run_id cur_batch_id cur_source; do
  [ -z "${cur_batch_id}" ] && continue
  
  audit_write_lineage "GOLD" "${RUN_ID}" "${BATCH_ID}" "${gold_table}" \
    "CURATED" "${cur_run_id}" "${cur_batch_id}" "${cur_source}"
done
```

Or via a single HiveQL INSERT:

```sql
INSERT INTO ${audit_db}.audit_lineage
SELECT
  'GOLD' AS layer,
  '${run_id}' AS run_id,
  '${batch_id}' AS batch_id,
  '${target_table}' AS target_table,
  'CURATED' AS source_layer,
  sc.run_id AS source_run_id,
  sc.batch_id AS source_batch_id,
  sc.source_name AS source_name,
  current_timestamp() AS created_ts
FROM ${audit_db}.audit_source_control sc
WHERE sc.layer = 'CURATED'
  AND sc.status = 'COMPLETED'
  AND sc.source_name IN (${curated_source_tables})  -- tables that feed this Gold table
  AND sc.watermark_end >= '${gold_watermark_start}'
  AND sc.watermark_start <= '${gold_watermark_end}';
```

**Notes:**
- The Curated side already wrote its CURATED→RAW edges in prompt 10.
- Raw's `audit_source_control` links batches to source files.
- DO NOT create any row-level lineage table. Cardinality of `audit_lineage` is
  batches × sources — small and manageable.

## 2. Row-level traceability by joining (document, don't materialize)

Create `docs/LINEAGE.md` showing the actual join chain using the real keys found in
prompt 14.

### The join chain

```
Gold row
  → _audit_batch_id 
  → audit_lineage (layer='GOLD', batch_id) 
  → Curated batch (source_batch_id)
  
Curated batch
  → audit_lineage (layer='CURATED', batch_id)
  → Raw batch (source_batch_id)
  
Raw batch
  → audit_source_control (layer='RAW', batch_id)
  → source_name (the Sequence File name)
  → MQ origin
```

### With business keys (for Raw, which has no audit columns)

```
Gold row (member_id, effective_dt)
  → _audit_batch_id → Curated batch
  
Curated row (matched by member_id, effective_dt, _audit_batch_id)
  → _audit_batch_id → Raw batch
  
Raw row (matched by member_id or transaction_header_id — from prompt 01/14)
  → The Raw layer has no _audit_batch_id, so join on business/natural key
  → audit_source_control (raw batch covering the date range)
  → source_file_name
```

### Worked example query

```sql
-- Given a Gold member row, trace back to the source file
-- Replace {gold_member_id} and {gold_effective_dt} with actual values

-- Step 1: Find Gold row's batch
SELECT _audit_batch_id 
FROM gold_member 
WHERE member_id = '{gold_member_id}' 
  AND effective_dt = '{gold_effective_dt}';
-- Returns: gold_batch_123

-- Step 2: Find Curated batch via lineage
SELECT source_run_id, source_batch_id, source_name
FROM audit_lineage
WHERE layer = 'GOLD' 
  AND batch_id = 'gold_batch_123';
-- Returns: curated_run_456, curated_batch_789, curated_member

-- Step 3: Find Raw batch via Curated lineage
SELECT source_run_id, source_batch_id, source_name
FROM audit_lineage
WHERE layer = 'CURATED' 
  AND batch_id = 'curated_batch_789';
-- Returns: raw_run_001, raw_batch_002, raw_member

-- Step 4: Find source file from Raw audit
SELECT source_name, source_path, file_modified_time
FROM audit_source_control
WHERE layer = 'RAW' 
  AND batch_id = 'raw_batch_002';
-- Returns: member_20240115.seq, /data/incoming/mq/member_20240115.seq, 2024-01-15 03:00:00
```

### One combined query (for convenience)

```sql
-- Full lineage trace from Gold to source file
WITH gold_row AS (
  SELECT _audit_batch_id AS gold_batch
  FROM gold_member
  WHERE member_id = '{member_id}' AND effective_dt = '{effective_dt}'
  LIMIT 1
),
gold_lineage AS (
  SELECT source_batch_id AS curated_batch
  FROM audit_lineage
  WHERE layer = 'GOLD' AND batch_id = (SELECT gold_batch FROM gold_row)
),
curated_lineage AS (
  SELECT source_batch_id AS raw_batch
  FROM audit_lineage
  WHERE layer = 'CURATED' AND batch_id = (SELECT curated_batch FROM gold_lineage)
)
SELECT 
  sc.source_name AS source_file,
  sc.source_path,
  sc.file_modified_time,
  sc.run_id AS raw_run_id
FROM audit_source_control sc
WHERE sc.layer = 'RAW' 
  AND sc.batch_id = (SELECT raw_batch FROM curated_lineage);
```

**Honest note**: The Raw→Curated hop for a specific row requires a business-key join
(Raw has no `_audit_batch_id` column — its schema is frozen). If the transaction-header
or natural key from prompt 01/14 is available, document its use here. Otherwise, lineage
at the Raw-row level is by batch containment, not exact row match.

## Completion gate input

Lineage rows written successfully is one of the Gold completion criteria (prompt 18).
After lineage INSERT:

```bash
lineage_count=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_lineage 
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
")

if [ ${lineage_count} -eq 0 ]; then
  fnLogMsg WARN "No lineage rows written for ${gold_table} — may indicate no Curated input found"
fi
```

## Deliverables

1. Shell logic to find and record Curated source batches
2. HiveQL INSERT for batch-level lineage
3. `docs/LINEAGE.md` with the full join chain and worked example
4. Lineage verification for completion gate

# G17 — Gold lineage (batch-level + traceability keys)

Implement lineage so any Gold record can be traced to its MQ-origin file.

## Baseline

Supersedes `prompts/17_Gold_Lineage.md`, which treated `${curated_source_tables}` as a value
passed in. It comes from `audit_gold_source_map` now.

## Inputs — read these before writing anything

- `docs/GOLD_ANALYSIS.md` (G14) — the natural keys connecting Gold rows back to Curated and
  Raw, needed for the join chain in part 2.
- `${AUDIT_DB}.audit_gold_source_map` — under fan-in a Gold table has N Curated inputs, so
  write **one edge per mapped source**. G18's completion gate compares the edge count against
  this map and fails the run if they disagree. Under fan-out the same Curated batch
  legitimately appears as the source of several Gold targets — expected, not duplication.

**Lineage is batch-level. Never build a row-level lineage table.** Cardinality of
`audit_lineage` is batches × sources — small and manageable.

## 1. Batch-level edges

Add after the SCD apply succeeds, before closing the batch:

```sql
INSERT INTO ${audit_db}.audit_lineage
SELECT
  'GOLD'            AS layer,
  '${run_id}'       AS run_id,
  '${batch_id}'     AS batch_id,
  '${gold_table}'   AS target_table,
  'CURATED'         AS source_layer,
  sc.run_id         AS source_run_id,
  sc.batch_id       AS source_batch_id,
  sc.source_name    AS source_name,
  current_timestamp() AS created_ts
FROM ${audit_db}.audit_source_control sc
JOIN ${audit_db}.audit_gold_source_map m
  ON sc.source_name = m.curated_table
WHERE m.gold_table = '${gold_table}'
  AND m.is_active   = 'Y'
  AND sc.layer      = 'CURATED'
  AND sc.status     = 'COMPLETED'
  AND sc.watermark_end   >= '${gold_watermark_start}'
  AND sc.watermark_start <= '${gold_watermark_end}';
```

The join to `audit_gold_source_map` is what makes this fan-in-correct — it picks up every
mapped input rather than a single hardcoded source table.

### Verify before moving on

```bash
expected_edges=$(audit_get_count "
  SELECT COUNT(DISTINCT curated_table) FROM ${AUDIT_DB}.audit_gold_source_map
  WHERE gold_table='${gold_table}' AND is_active='Y'")

actual_edges=$(audit_get_count "
  SELECT COUNT(DISTINCT source_name) FROM ${AUDIT_DB}.audit_lineage
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'")

if [ ${actual_edges} -lt ${expected_edges} ]; then
  fnLogMsg WARN "Lineage incomplete for ${gold_table}: ${actual_edges}/${expected_edges} sources"
fi
```

A source with zero completed Curated batches in the window produces no edge. That should not
happen — G15's fan-in gate already refused to build the target in that case. If you see it,
the gate and this query are using **different window bounds**. Check both.

Notes:

- The Curated side already wrote its CURATED→RAW edges (`prompts/10`, patched in R05).
- Raw's `audit_source_control` links batches to source files.

## 2. Row-level traceability by joining (document, do not materialize)

Create `docs/LINEAGE.md` using the **real** keys from `docs/GOLD_ANALYSIS.md`.

### The join chain

```
Gold row      → _audit_batch_id → audit_lineage (layer='GOLD')     → Curated batch(es)
Curated batch → audit_lineage (layer='CURATED')                    → Raw batch
Raw batch     → audit_source_control (layer='RAW')                 → source file → MQ origin
```

### Worked example query

```sql
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
  WHERE layer = 'CURATED' AND batch_id IN (SELECT curated_batch FROM gold_lineage)
)
SELECT sc.source_name AS source_file, sc.source_path,
       sc.file_modified_time, sc.run_id AS raw_run_id
FROM audit_source_control sc
WHERE sc.layer = 'RAW'
  AND sc.batch_id IN (SELECT raw_batch FROM curated_lineage);
```

Replace `gold_member`, `member_id` and `effective_dt` with real names — they are placeholders.

### Two honest caveats to state in that file

1. **A Gold row built by fan-in traces back to several Curated batches, not one.** The trace
   answers *"which batches could have contributed"*, not *"which batch produced this exact
   column value"*. Note the `IN` rather than `=` in the query above. Narrowing further needs
   the business key.
2. **The Raw hop is by batch containment, not exact row match.** Raw's schema is frozen so it
   has no `_audit_batch_id`; join on the natural key from `prompts/01` / G14. If no usable
   natural key exists, say so plainly rather than implying a precision the data cannot give.

## Deliverables

1. The lineage INSERT, joined through `audit_gold_source_map`.
2. The edge-count verification.
3. `docs/LINEAGE.md` with the join chain, a worked query using real keys, and both caveats.
4. Confirmation that no row-level lineage table was created.

# Prompt 17a — Add Gold lineage

Record which Curated batches produced each Gold batch, so any Gold row can be traced back
to its source file.

**Context:**
- Gold run/source audit is in place (from 15a)
- `${AUDIT_DB}.audit_gold_source_map` is loaded (from 14a)
- The Curated side already wrote its CURATED→RAW edges (from 10b)
- Natural keys for the join chain come from `docs/GOLD_ANALYSIS.md`

**Lineage is batch-level. Never build a row-level lineage table.** Cardinality of
`audit_lineage` is batches × sources — small. Row-level tracing works by joining
`_audit_batch_id` plus business keys, documented at the bottom of this prompt.

## Write one edge per mapped Curated source

Not one edge per Gold table. Under fan-in a Gold table has N Curated inputs, and the 18a
completion gate compares the edge count against `audit_gold_source_map` — writing a single
edge fails the gate. Under fan-out the same Curated batch appears as the source of several
Gold targets; that is expected, not duplication.

**Add after the SCD apply succeeds, before closing the batch:**

```bash
# ============ LINEAGE ============
# One row per (Curated source table × contributing Curated batch) for this Gold batch

lineage_hql="
INSERT INTO ${AUDIT_DB}.audit_lineage
SELECT
  'GOLD'            AS layer,
  '${RUN_ID}'       AS run_id,
  '${BATCH_ID}'     AS batch_id,
  '${gold_table}'   AS target_table,
  'CURATED'         AS source_layer,
  sc.run_id         AS source_run_id,
  sc.batch_id       AS source_batch_id,
  sc.source_name    AS source_name,
  current_timestamp() AS created_ts
FROM ${AUDIT_DB}.audit_source_control sc
JOIN ${AUDIT_DB}.audit_gold_source_map m
  ON sc.source_name = m.curated_table
WHERE m.gold_table = '${gold_table}'
  AND m.is_active   = 'Y'
  AND sc.layer      = 'CURATED'
  AND sc.status     = 'COMPLETED'
  AND sc.watermark_end   >= '${watermark_start}'
  AND sc.watermark_start <= '${watermark_end}'
"

hivebeeline -e "${lineage_hql}"

# Verify: edges written should equal active sources for this Gold table
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

The join to `audit_gold_source_map` is what makes this fan-in-correct — it picks up every
mapped input rather than a single hardcoded source table.

A source with zero completed Curated batches in the window produces no edge. That should
not happen, because 15a's fan-in gate already refused to build the target in that case; if
you see it, the gate and this query are using different window bounds. Check both.

## Document the trace chain

Create `docs/LINEAGE.md` with the join chain, using the **real** keys from
`docs/GOLD_ANALYSIS.md`:

```
Gold row  → _audit_batch_id → audit_lineage (layer='GOLD')  → Curated batch(es)
Curated batch → audit_lineage (layer='CURATED')             → Raw batch
Raw batch → audit_source_control (layer='RAW')              → source file → MQ origin
```

**Two honest caveats to state in that file:**

1. A Gold row built by fan-in traces back to **several** Curated batches, not one. The
   trace answers "which batches could have contributed", not "which batch produced this
   exact column value". Narrowing further needs the business key.
2. The Raw hop is by **batch containment, not exact row match** — Raw's schema is frozen so
   it has no `_audit_batch_id`. Join on the natural key from prompt 01/14a.

Include one worked query that goes from a real Gold key to a source file name.

[PASTE: Your Gold script section where the SCD apply completes]

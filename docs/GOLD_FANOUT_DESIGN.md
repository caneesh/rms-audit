# Gold Fan-In / Fan-Out Design

How the audit framework handles the many-to-many relationship between Curated and Gold.
Read this after `AUDIT_DEVELOPER_GUIDE.md` and before running prompts 14–18.

---

## 1. The problem

The developer guide describes Curated → Gold as "1-to-many". It is actually **many-to-many**,
and both directions break the naive instrumentation:

- **Fan-out** — one Curated table feeds several Gold tables. `curated_member` may land in
  `gold_member_dim`, `gold_member_coverage`, and `gold_member_addr_hist`, each with a
  different filter and a different cardinality.
- **Fan-in** — one Gold table is assembled from several Curated tables. `gold_member_dim`
  may read `curated_member` for its row population plus `curated_member_addr` and
  `curated_plan` for enrichment columns.

Two consequences:

1. **Reconciliation has no identity at layer grain.** "curated_member → GOLD" is not a
   checkable statement, because the same source row legitimately produces rows in three
   targets under three different rules.
2. **Fan-in can fail silently.** If four of five Curated inputs completed, the Gold table
   still builds, still has a plausible row count, and is quietly wrong — stale enrichment
   with no count mismatch to catch it.

---

## 2. The core decision: batch = one Gold target build

Define:

```
BATCH_ID = f(RUN_ID, gold_table)
```

One batch per Gold table per run — **not** one batch per source read. Everything else
follows from this.

| Table | Rows per Gold batch |
|---|---|
| `audit_source_control` | **N** — one per Curated input, each with its own window and `input_count` |
| `audit_stage_summary` | 1 — `target_table` stays unambiguous |
| `audit_merge_summary` | 1 — the SCD/MERGE apply for that target |
| `audit_lineage` | **N** — one edge per contributing Curated batch |
| `audit_reconciliation` | 1 per DRIVER edge (see section 4) |

The decisive reason is the `_audit_batch_id` column carried on the Gold row. A row
assembled from five Curated tables still has exactly **one** batch id stamped on it, and
all five sources are recovered by joining to `audit_lineage`. Under a source-scoped batch
grain that column would be unanswerable.

No DDL change is needed for this. `batch_id` was never unique in `audit_source_control`;
the fix is instrumentation discipline.

---

## 3. `audit_lineage` is the many-to-many bridge

No new lineage structure is required. The existing edge table resolves both directions:

```
audit_lineage(layer, run_id, batch_id, target_table,
              source_layer, source_run_id, source_batch_id, source_name)
```

- **Fan-in**  — many rows sharing `batch_id` / `target_table`, differing by `source_name`.
- **Fan-out** — one `source_batch_id` appearing in many rows with different `target_table`.

Cardinality stays small (batches × sources). The rule from prompt 17 holds unchanged:
**never build a row-level lineage table.**

---

## 4. Declare the mapping as data, with a role per source

The Curated→Gold map lives in `audit_gold_source_map`, not scattered through shell.
Prompt 14's analysis produces it.

| gold_table | curated_table | source_role | join_key | expected_cardinality | depends_on |
|---|---|---|---|---|---|
| gold_member_dim | curated_member | DRIVER | member_id | 1:1 on key | |
| gold_member_dim | curated_member_addr | ENRICH | member_id | N:1 | |
| gold_member_dim | curated_plan | LOOKUP | plan_cd | N:1 | |
| gold_member_coverage | curated_coverage | DRIVER | coverage_id | 1:1 on key | gold_member_dim |

`source_role` is what dissolves fan-in. When five Curated tables feed one Gold table, one
is normally the **driver** — its row population determines the Gold row population — and
the rest add columns, not rows. So:

- **DRIVER** sources get a reconciliation identity.
- **ENRICH / LOOKUP** sources get **referential-integrity rules** instead
  (`rule_type='RI'`, prompt 16). "Every driver row matched a `plan_cd`" is the right
  question for a lookup; asking it to reconcile by count is meaningless.

All roles still get an `audit_source_control` row and a lineage edge, so completeness and
traceability cover every input. Only the *accounting identity* is driver-scoped.

`depends_on` records Gold→Gold ordering: `gold_member_coverage`'s RI check joins
`gold_member_dim`, so the dim must build first. The table loop walks the map in dependency
order.

---

## 5. Reconcile per edge, on distinct natural keys

### Per edge, never per layer

`audit_reconciliation` gains `source_name` and `target_table`. One row per
(Curated table → Gold table) edge, each with its own `expected_difference_reason`:

| edge | expected relationship |
|---|---|
| curated_member → gold_member_dim | 1:1 on member key |
| curated_member → gold_member_coverage | only members with active coverage; filter reason recorded |
| curated_member → gold_member_addr_hist | SCD2; version rows expected to exceed keys |

**Never sum across edges.** The tempting-but-wrong assertion is that a Curated table's row
count equals the total across all Gold targets it feeds. Each edge is an independent
identity and is evaluated alone.

### The identity is in keys, not rows

```
distinct_source_keys_in_scope
    = distinct_target_keys_touched + rejected_keys + filtered_keys
```

Row counts break under both directions — SCD2 inflates the target side with version rows,
and an enrichment table with multiple rows per key inflates the source side. Distinct
natural keys are stable under both.

`distinct_target_keys_touched` comes from `audit_merge_summary`
(`inserted_count + updated_count`), which counts key operations rather than physical rows.
As in prompt 18, all counts come from audit rows already written during the run — **no new
table scans.**

`unexplained_difference != 0` → `MISMATCHED` → the Gold run fails.

---

## 6. Fan-in gate: all inputs ready before building

Run this per Gold target, **before** any build. This is the check that prevents the silent
staleness described in section 1.

```bash
missing_inputs=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_gold_source_map m
  WHERE m.gold_table = '${gold_table}'
    AND NOT EXISTS (
      SELECT 1 FROM ${AUDIT_DB}.audit_source_control sc
      WHERE sc.layer = 'CURATED'
        AND sc.status = 'COMPLETED'
        AND sc.source_name = m.curated_table
        AND sc.watermark_end   >= '${window_start}'
        AND sc.watermark_start <= '${window_end}')")

if [ ${missing_inputs} -gt 0 ]; then
  missing_list=$(audit_get_value "SELECT CONCAT_WS(',', COLLECT_LIST(m.curated_table)) ...")
  audit_partial_source "${RUN_ID}" "${BATCH_ID}" \
    "Missing Curated inputs: ${missing_list}"
  fnLogMsg WARN "Skipping ${gold_table}: missing inputs ${missing_list}"
  continue
fi
```

Skip the target, name the missing inputs, and let the run continue with other targets
rather than failing wholesale. **Partial-but-honest beats complete-but-silently-stale.**

---

## 7. Instrumentation shape

```bash
for gold_table in $(gold_targets_in_dependency_order); do
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${gold_table}")

  # 6. fan-in gate — all mapped inputs COMPLETED for this window?
  # ... (section 6) ...

  # one source_control row per mapped Curated input, all sharing BATCH_ID
  for src in $(gold_sources_for "${gold_table}"); do
    input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${src} WHERE ${watermark_condition}")
    audit_start_source "GOLD" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${src}" \
      "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
  done

  # ========== EXISTING SCD/MERGE PROCESSING — UNCHANGED ==========

  # one stage_summary + one merge_summary for the batch
  # one lineage edge per contributing Curated batch
  # one reconciliation row per DRIVER edge
done
```

---

## 8. Completion gate

Prompt 18's gate checks only that lineage rows *exist* per target. Under fan-in that is
too weak — one edge written out of five passes. Compare the edge count against the map:

```sql
-- Gold targets whose lineage edges don't match their declared source count
SELECT l.target_table
FROM (SELECT target_table, COUNT(*) AS c
      FROM ${audit_db}.audit_lineage
      WHERE layer = 'GOLD' AND run_id = '${run_id}'
      GROUP BY target_table) l
JOIN (SELECT gold_table, COUNT(*) AS c
      FROM ${audit_db}.audit_gold_source_map
      GROUP BY gold_table) m
  ON l.target_table = m.gold_table
WHERE l.c <> m.c;
```

Non-zero → `lineage_incomplete` → run FAILED.

---

## 9. What does not change

These are unaffected once batch means "one Gold target build" — the signal that the grain
choice is right:

- `audit_lineage` — structure unchanged; it was already an edge table.
- `audit_stage_summary`, `audit_merge_summary` — one row per target, as before.
- `_audit_run_id` / `_audit_batch_id` on Gold rows — still single-valued.
- The five rules in the developer guide, including append-only and the kill switch.
- Row-level lineage is still never materialized.

---

## 10. Where this lands in the prompts

| Prompt | Change |
|---|---|
| 02 — Unified DDL | `source_name` + `target_table` on `audit_reconciliation`; new `audit_gold_source_map` table |
| 14 — Analyze Gold | Curated→Gold map with `source_role` per source becomes a required deliverable |
| 15 / 15a — Gold run+source audit | Nested loop, target-scoped `BATCH_ID`, fan-in readiness gate |
| 18 — Reconciliation | Per-edge reconciliation on distinct keys; stricter lineage gate |

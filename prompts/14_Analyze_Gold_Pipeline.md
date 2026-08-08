# Prompt 14 — Analyze the Gold pipeline (shell/beeline)

Analyze the Gold pipeline implementation. **Do not make any code changes.**

Like Curated, this pipeline runs as shell scripts calling `hivebeeline`.

## Shell orchestration

Identify and report:
1. Entry point script(s): `gold_mbrshp_rms_*.sh` — which is the master, which are helpers?
2. Script call graph: which scripts call which, in what order.
3. How table iteration works (similar to Curated's `$goldTablesFromParams`?).
4. Trigger/dependency mechanism: how does Gold know Curated is complete?
5. Where `.prm` parameter files live and key variables they define.
6. Where `.hql` files live for Gold transformations.
7. Existing error handling and logging patterns.

## Data flow (per Gold table)

8. **The Curated → Gold mapping.** This is the load-bearing deliverable of this prompt —
   everything in prompts 15–18 keys off it. The relationship is **many-to-many**, not
   1-to-many: one Gold table is assembled from several Curated tables (fan-in) *and* one
   Curated table feeds several Gold tables (fan-out). Read
   `docs/GOLD_FANOUT_DESIGN.md` before answering this one.

   Produce one row per (gold_table, curated_table) edge:

   | gold_table | curated_table | source_role | join_key | expected_cardinality | depends_on |
   |---|---|---|---|---|---|
   | gold_member_dim | curated_member | DRIVER | member_id | 1:1 on key | |
   | gold_member_dim | curated_plan | LOOKUP | plan_cd | N:1 | |

   - `source_role` — **DRIVER** if that source's row population determines the Gold
     table's row population; **ENRICH** if it adds columns to driver rows; **LOOKUP** if
     it is a dimension/code join. Most Gold tables have exactly one DRIVER. If you find a
     Gold table with two or more genuine drivers (a union or a full outer join), flag it
     explicitly — it needs a hand-written reconciliation identity.
   - `expected_cardinality` — what the join is *supposed* to do (1:1 on key, N:1, 1:N).
     A LOOKUP that turns out to be 1:N is a row-multiplication bug waiting to happen; say so.
   - `depends_on` — another Gold table that must be built first, e.g. because this table's
     RI checks join it.

   This table becomes the seed data for `audit_gold_source_map` (prompt 02).

9. How history is maintained: SCD type per table (1/2/other), PIT tables if any.
10. Effective/expiry date handling: how are versions opened and closed?
11. How Gold writes physically happen: partition overwrite? append? MERGE?

## Business logic

12. Key business rules: current coverage, latest member, primary address, subscriber
    selection, active product, effective-date logic — name what exists in the HQL files.
13. Referential-integrity relationships: member exists, coverage exists, provider exists,
    product exists — identify the actual foreign keys used.
14. Every intentional drop/filter/dedup (named reconciliation reasons).

## Schema and traceability

15. Whether Gold schemas can accept `_audit_run_id`, `_audit_batch_id` columns.
16. Natural keys that connect a Gold row back to Curated and Raw rows (member_id,
    transaction_header_id, etc.) — needed for row-level traceability via JOINs.
17. Any existing run/batch tracking today.

## Specific scripts to analyze (from the file listing)

- `gold_mbrshp_rms_raw_load.sh` — what does this do in Gold context?
- `gold_mbrshp_rms_cdckeys_load.sh` / `write_cdckeys.sh` / `delete_cdc_keys.sh`
- `gold_mbrshp_rms_common_hiveMerge_withGoldDB.sh` — the merge utility
- `gold_mbrshp_rms_hiveMerge_CDCchanges_withCurrentDB.sh`
- `gold_mbrshp_rms_history_table.sh` / `history_table_load.sh`
- `gold_mbrshp_rms_create_cdc_trigger.sh` / `create_gcf_trg.sh` / `create_gld_strt_trg.sh`
- `gold_mbrshp_rms_create_check_delete_stoppers.sh`

## Output

- Shell script inventory with call graph and purpose of each
- **The Curated→Gold edge table from item 8** — one row per (gold_table, curated_table)
  with `source_role`, `join_key`, `expected_cardinality`, `depends_on`. This seeds
  `audit_gold_source_map`.
- **Per DRIVER edge: the expected source→target relationship in one sentence**, phrased in
  distinct natural keys rather than row counts — e.g. "distinct member keys in
  curated_member for the window = member keys touched in gold_member_dim". This becomes
  `expected_difference_reason` in prompt 18. Fan-out means each edge gets its own
  sentence; never one per Curated table.
- Per-Gold-table: SCD type, HQL file locations, build order implied by `depends_on`
- Rule inventory from HQL files
- RI relationships and their keys — note which ones correspond to ENRICH/LOOKUP edges,
  since those are audited as RI rules instead of reconciliation
- Drop/filter inventory with reasons
- Schema-change feasibility
- Traceability key mapping
- Risks and integration points for audit instrumentation

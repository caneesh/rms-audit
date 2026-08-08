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

8. The Curated → Gold mapping: which Curated tables feed which Gold tables.
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
- Per-Gold-table: Curated sources, SCD type, HQL file locations
- Rule inventory from HQL files
- RI relationships and their keys
- Drop/filter inventory with reasons
- Schema-change feasibility
- Traceability key mapping
- Risks and integration points for audit instrumentation

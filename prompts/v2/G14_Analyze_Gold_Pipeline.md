# G14 — Analyze the Gold pipeline (shell/beeline)

Analyze the Gold pipeline implementation. **Do not make any code changes.**

## Baseline

Supersedes `prompts/14_Analyze_Gold_Pipeline.md`. Read `docs/GOLD_FANOUT_DESIGN.md` first —
it explains why item 8 below is the load-bearing deliverable of this prompt.

**This is a hard gate.** G15–G18 read the two files this prompt writes. They will not have
this conversation, so anything left in chat is lost.

Like Curated, this pipeline runs as shell scripts calling `hivebeeline`.

## Shell orchestration

1. Entry point script(s): `gold_mbrshp_rms_*.sh` — which is the master, which are helpers?
2. Script call graph: which scripts call which, in what order.
3. How table iteration works (similar to Curated's `$goldTablesFromParams`?).
4. Trigger/dependency mechanism: how does Gold know Curated is complete?
5. Where `.prm` parameter files live and key variables they define.
6. Where `.hql` files live for Gold transformations.
7. Existing error handling and logging patterns.

## Data flow (per Gold table)

### 8. The Curated → Gold mapping — the most important answer here

The relationship is **many-to-many**, not 1-to-many: one Gold table is assembled from several
Curated tables (fan-in) *and* one Curated table feeds several Gold tables (fan-out).

Produce one row per (gold_table, curated_table) edge:

| gold_table | curated_table | source_role | join_key | expected_cardinality | depends_on |
|---|---|---|---|---|---|

- **`source_role`** — **DRIVER** if that source's row population determines the Gold table's
  row population (usually the table in the `FROM` clause of the main SELECT); **ENRICH** if
  it adds columns to driver rows; **LOOKUP** if it is a code/dimension join. Most Gold tables
  have exactly one DRIVER. If you find one with two or more genuine drivers — a UNION or a
  FULL OUTER JOIN — **flag it explicitly**; it needs a hand-written reconciliation identity.
- **`expected_cardinality`** — what the join is *supposed* to do (1:1 on key, N:1, 1:N). A
  LOOKUP that turns out to be 1:N is a row-multiplication bug waiting to happen; say so.
- **`depends_on`** — another Gold table that must be built first, e.g. because this table's
  RI checks join it.

9. How history is maintained: SCD type per table (1/2/other), PIT tables if any.
10. Effective/expiry date handling: how are versions opened and closed?
11. How Gold writes physically happen: partition overwrite? append? MERGE?

## Business logic

12. Key business rules: current coverage, latest member, primary address, subscriber
    selection, active product, effective-date logic — name what exists in the HQL files.
13. Referential-integrity relationships: member exists, coverage exists, provider exists,
    product exists — identify the actual foreign keys used. Note which correspond to
    ENRICH/LOOKUP edges from item 8, since those are audited as RI rules rather than
    reconciliation.
14. Every intentional drop/filter/dedup (named reconciliation reasons).

## Schema and traceability

15. Whether Gold schemas can accept `_audit_run_id`, `_audit_batch_id` columns.
16. Natural keys that connect a Gold row back to Curated and Raw rows (member_id,
    transaction_header_id, etc.) — needed for row-level traceability via JOINs.
17. Any existing run/batch tracking today.

## 18. Environment facts that G15–G18 hardcode

Those prompts contain placeholders that must be replaced with this pipeline's real values.
Report each with the **file and line** you found it in, and say **NOT FOUND** rather than
guessing — a guessed value produces instrumentation that runs and writes wrong rows.

| Placeholder | What to find |
|---|---|
| `${goldTablesFromParams}` | the real variable holding the Gold table list, and where it is set |
| `${hiveDB_gold}` / `${hiveDB_curated}` | actual Hive database names |
| `${AUDIT_DB}` | decide and state it; it must not vary between prompts |
| `${AUDIT_LIB_PATH}`, `${AUDIT_HQL_PATH}` | where the audit library and its HQL live |
| `${watermark_condition}` | the literal SQL predicate selecting a Curated processing window |
| `${CURATED_WATERMARK_START/END}` | how window bounds are supplied — trigger file, `.prm`, derived |
| `${current_date}`, `load_date` | the real partition column name and format on Gold tables |
| `hivebeeline` | exact wrapper name and the flags it already passes |
| `fnLogMsg` | exact signature, and whether it is sourced or defined inline |
| `${BLOCKING_RULES}`, `${REJECTING_RULES}`, `${FILTERING_RULES}` | which rule failures stop a load, which reject rows, which are intentional filters — ask if unclear, do not invent |
| SCD action column | the column the SCD logic sets (`scd_action` in G16 is a placeholder) |
| Hive / Spark version | whether `MERGE INTO` and window functions are available |

## 19. Natural key per Gold entity

G18 reconciles on **distinct natural keys, not rows**, so it needs the actual key column(s)
per entity.

## 20. Expected relationship per DRIVER edge

One sentence each, phrased in distinct keys — e.g. *"distinct member keys in curated_member
for the window = member keys touched in gold_member_dim"*. This becomes
`expected_difference_reason` in G18.

Fan-out means each **edge** gets its own sentence; never one per Curated table. This is the
one thing in the whole pack that **cannot be derived from the code** — it is a business
statement about what the Gold table is supposed to contain. If you cannot determine it, say
so and ask, rather than inferring one from observed counts.

## Specific scripts to analyze (from the file listing)

- `gold_mbrshp_rms_raw_load.sh` — what does this do in Gold context?
- `gold_mbrshp_rms_cdckeys_load.sh` / `write_cdckeys.sh` / `delete_cdc_keys.sh`
- `gold_mbrshp_rms_common_hiveMerge_withGoldDB.sh` — the merge utility
- `gold_mbrshp_rms_hiveMerge_CDCchanges_withCurrentDB.sh`
- `gold_mbrshp_rms_history_table.sh` / `history_table_load.sh`
- `gold_mbrshp_rms_create_cdc_trigger.sh` / `create_gcf_trg.sh` / `create_gld_strt_trg.sh`
- `gold_mbrshp_rms_create_check_delete_stoppers.sh`

## Output — two files, not chat

### `docs/GOLD_ANALYSIS.md`

- Shell script inventory with call graph and purpose of each
- **The Curated→Gold edge table from item 8**
- **Per DRIVER edge: the expected relationship sentence from item 20**
- **The environment-facts table from item 18, with real values filled in**
- **The natural key per entity from item 19**
- Per-Gold-table: SCD type, HQL file locations, build order implied by `depends_on`
- Rule inventory from the HQL files
- RI relationships and their keys, flagged against ENRICH/LOOKUP edges
- Drop/filter inventory with reasons
- Schema-change feasibility
- Traceability key mapping
- Risks and integration points for audit instrumentation

### `sql/audit_gold_source_map_seed.sql`

INSERT statements populating `${AUDIT_DB}.audit_gold_source_map` from the item-8 table, ready
to run. Set `is_active='Y'` for every current edge.

Load it into the audit database before running G15 — every Gold loop reads it, and an empty
map means the instrumentation silently does no work.

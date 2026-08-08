# R02 — Fix the audit DDL

**Part of the R02–R05 atomic block. Do not run any pipeline until R05 is complete.**

## Baseline

Supersedes `prompts/02_Create_Unified_Audit_Schema_DDL.md`. That prompt's global rules,
naming, storage format and partitioning guidance all still apply — only the two changes
below are new. Use `docs/AUDIT_V2_BASELINE.md` (R01) for the real file paths and for what is
actually deployed in Hive.

The audit tables hold **test data only**, so drop-and-recreate is approved. If R01 found data
anyone needs, stop and raise it — this prompt would need reworking as `ALTER TABLE`.

## Change 1 — `audit_reconciliation` gains two columns

### Why

Curated→Gold is many-to-many (see `docs/GOLD_FANOUT_DESIGN.md`). One Curated table feeds
several Gold tables under different rules, so `(entity_name, from_layer, to_layer)` does not
identify a reconciliation row — three rows for `curated_member` would be indistinguishable.
Reconciliation has to be recorded per **edge**: one source table to one target table.

### What

Add `source_name` and `target_table` immediately after `to_layer`:

```
layer, run_id, entity_name, from_layer, to_layer,
source_name STRING,      -- source table for this edge
target_table STRING,     -- target table for this edge
source_count BIGINT, target_count BIGINT, rejected_count BIGINT, filtered_count BIGINT,
difference BIGINT, expected_difference_reason STRING, unexplained_difference BIGINT,
status STRING,           -- MATCHED | EXPLAINED | MISMATCHED
created_ts TIMESTAMP
```

Position matters: `audit_write_reconciliation` and the reconciliation HQL both insert
positionally, and R03/R04/R05 assume this order.

Add a column comment to `source_count` and `target_count` stating that they are **distinct
natural keys, not physical row counts**. Row counts break under SCD2 (version rows inflate
the target) and under fan-in enrichment (multiple rows per key inflate the source).

### How

Drop and recreate the table, then regenerate the master DDL file so it stays the single
source of truth. Keep the commented-out DROP as a rollback example, per the original prompt's
convention.

## Change 2 — new table `audit_gold_source_map`

### Why

The Gold track (G14–G18) drives its loops from this mapping instead of hardcoding which
Curated tables feed which Gold table. It is what makes fan-in and fan-out tractable.

### What

```
gold_table STRING,
curated_table STRING,
source_role STRING,            -- DRIVER | ENRICH | LOOKUP
join_key STRING,
expected_cardinality STRING,   -- e.g. '1:1 on key', 'N:1'
depends_on STRING,             -- nullable: Gold table that must be built first
is_active STRING,              -- 'Y' | 'N'
created_ts TIMESTAMP
```

**This table is the exception to two global rules in `prompts/02`.** It is reference data,
not events: it has **no `layer` column** and is **not append-only** — it is replaced when the
Curated→Gold mapping changes. Say both things in the table comment, so nobody "fixes" it
later to match the other eight.

Column comments must explain:

- `source_role` — **DRIVER** sources determine the target's row population and get a
  reconciliation identity in G18. **ENRICH** and **LOOKUP** sources add columns rather than
  rows, so counting them is meaningless; they get referential-integrity rules in G16 instead.
  Every role still gets an `audit_source_control` row and an `audit_lineage` edge.
- `depends_on` — drives Gold build ordering, for when one Gold table's RI checks join another.

Do not partition this table; it is small.

It is created **empty**. G14 produces `sql/audit_gold_source_map_seed.sql` from its analysis
of the real pipeline, and that is what populates it. Do not invent rows here.

## Deliverables

1. Updated per-table DDL for `audit_reconciliation`, and a new one for
   `audit_gold_source_map`.
2. Regenerated master DDL file, with `audit_gold_source_map` in creation order.
3. The drop-and-recreate script actually run against the audit database.
4. `DESCRIBE FORMATTED` output for both tables, confirming the deployed schema matches.
5. A one-line note in the DDL header recording why `audit_gold_source_map` is exempt from
   the `layer` column and the append-only model.

## Do not stop here

R03 must follow immediately — the Scala models now disagree with this schema, and any
model-vs-DDL drift test is failing as of this commit.

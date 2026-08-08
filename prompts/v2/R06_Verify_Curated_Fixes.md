# R06 — Verify the Curated fixes

R02–R05 changed a schema, a Scala model, three shell signatures and every Curated call site.
Two of those defects — the double-count and the positional-argument shift — produce
**plausible-looking wrong data rather than errors**, so reading the diff proves nothing. This
prompt proves the repair by running it.

## Baseline

Requires R02–R05 all applied. Use `docs/AUDIT_V2_BASELINE.md` (R01) section 5 for the
"before" numbers.

## Before running

Confirm the block is complete:

- [ ] `audit_reconciliation` deployed with `source_name` and `target_table` (R02)
- [ ] `audit_gold_source_map` exists and is empty (R02)
- [ ] Scala compiles; drift test passes or is recorded as absent (R03)
- [ ] `audit_functions.sh` has the new signatures and the five new functions (R04)
- [ ] Every call site from R04's enumeration updated (R05)

## Run

Pick one Curated table with a known, modest input volume and run the pipeline for a single
window with `AUDIT_ENABLED=true`. Record the `RUN_ID` and `BATCH_ID`.

## Assertions

### 1. Reconciliation is clean — the headline check

```sql
SELECT entity_name, source_name, target_table,
       source_count, target_count, rejected_count, filtered_count,
       unexplained_difference, expected_difference_reason, status
FROM ${AUDIT_DB}.audit_reconciliation
WHERE layer='CURATED' AND run_id='${RUN_ID}';
```

Expect `unexplained_difference = 0` and status `MATCHED` or `EXPLAINED`. `source_name` and
`target_table` must both be non-null and hold **table names**, not numbers — a number in
`source_name` means a call site still passes the old argument order.

If R01 recorded a non-zero `unexplained_difference` for a comparable batch, show the
before/after side by side. That contrast is the proof the double-count is gone.

If `source_count` is still roughly double what the source table actually holds for the
window, Fix 1 was not applied everywhere — sweep again.

### 2. Source rows read correctly

```sql
SELECT batch_id, source_name, status, input_count, processed_count, created_ts
FROM ${AUDIT_DB}.audit_source_control
WHERE layer='CURATED' AND run_id='${RUN_ID}'
ORDER BY created_ts;
```

Expect at least two rows per source (STARTED then COMPLETED) — that is the append-only model
working, not a bug. `source_name` must hold a table name on **every** row including the
COMPLETED one, which is what proves `audit_complete_source`'s new third argument is wired
correctly.

### 3. Lineage carries a source run id

```sql
SELECT target_table, source_layer, source_run_id, source_batch_id, source_name
FROM ${AUDIT_DB}.audit_lineage
WHERE layer='CURATED' AND run_id='${RUN_ID}';
```

Every row must have a non-empty `source_run_id`. Empty means Fix 4 was missed.

### 4. End-of-run tallies agree

```sql
SELECT status, total_sources, completed_sources, failed_sources, error_message
FROM ${AUDIT_DB}.audit_run_control
WHERE run_id='${RUN_ID}' ORDER BY created_ts DESC LIMIT 1;
```

`completed_sources + failed_sources` must not exceed `total_sources`, and `total_sources`
must equal the number of distinct sources actually processed.

### 5. Append-only intact

```bash
grep -rniE '\bupdate\b[[:space:]]+[a-z_.]*audit_' <audit code paths>
```

Must return nothing. Also confirm the row counts in `audit_source_control` for this run only
grew — no row was replaced.

### 6. Kill switch still restores today's behaviour

Re-run the same window with `AUDIT_ENABLED=false`. The business output must be identical, and
no new audit rows may appear for that run. This is the rollback plan; it has to keep working
after four prompts of changes.

## If an assertion fails

Report which one, with the actual query output. Do **not** patch the audit tables by hand to
make a check pass — the tables are the evidence. Fix the code and re-run.

## Deliverables

1. The `RUN_ID` / `BATCH_ID` used, and the table chosen.
2. Actual output for each of the six assertions — not a summary, the rows.
3. The before/after reconciliation comparison if R01 captured a "before".
4. A clear verdict: is Curated correct, and is it safe to start G14.

Do not start the Gold track until every assertion above passes.

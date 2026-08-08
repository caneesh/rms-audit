# G15 — Instrument Gold shell scripts: run, source and entity audit

Add run/source/stage audit to the Gold pipeline shell scripts using the audit library.
Layer='GOLD'. Kill switch applies.

Do not change HiveQL transformations or business logic.

## Baseline

Supersedes `prompts/15_Gold_Run_Source_And_Entity_Audit.md`, which assumed one Curated source
per Gold target. That is wrong — see `docs/GOLD_FANOUT_DESIGN.md`.

## Inputs — read these before writing anything

1. `docs/GOLD_ANALYSIS.md` (G14) — script inventory, the Curated→Gold edge map, and the
   **real values** for every placeholder below.
2. `${AUDIT_DB}.audit_gold_source_map` — must already be loaded from
   `sql/audit_gold_source_map_seed.sql`, since the loops below read it.
3. `docs/GOLD_FANOUT_DESIGN.md` — why the batch grain is what it is.
4. The audit shell library after R04, including `audit_partial_source` and the `audit_gold_*`
   helpers.

If any of these is missing, or a placeholder is unresolved, **stop and say so**. Do not
substitute a plausible-looking name.

## The batch grain — the decision everything else follows from

**One batch per Gold target table, not per source read.** A Gold table assembled from five
Curated inputs produces five `audit_source_control` rows sharing one `BATCH_ID`, and one
`audit_stage_summary` row. That is what keeps `_audit_batch_id` on the Gold row
single-valued: a row built from five sources still carries exactly one batch id, and the five
sources are recovered through `audit_lineage`.

## At pipeline start

```bash
source ${AUDIT_LIB_PATH}/audit_functions.sh
audit_init || { fnLogMsg ERROR "Audit init failed"; }

RUN_ID=$(audit_generate_run_id)
audit_start_run "GOLD" "${RUN_ID}" "${PIPELINE_NAME}" "${PROCESS_NAME}" "${SOURCE_SYSTEM}"
```

## Per Gold target: batch id, fan-in gate, then one source row per input

Iterate targets in **dependency order**, and generate the batch id from the *target*:

```bash
for gold_table in $(audit_gold_targets_in_dependency_order); do
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${gold_table}")

  watermark_start="${CURATED_WATERMARK_START}"
  watermark_end="${CURATED_WATERMARK_END}"
```

### Fan-in readiness gate — before any build

If a Gold table needs five Curated inputs and only four completed, the build still produces a
plausible row count and is **quietly wrong** — stale enrichment with no count mismatch to
catch it. Check every mapped input first:

```bash
  missing_inputs=$(audit_get_count "
    SELECT COUNT(*) FROM ${AUDIT_DB}.audit_gold_source_map m
    WHERE m.gold_table='${gold_table}' AND m.is_active='Y'
      AND NOT EXISTS (
        SELECT 1 FROM ${AUDIT_DB}.audit_source_control sc
        WHERE sc.layer='CURATED' AND sc.status='COMPLETED'
          AND sc.source_name = m.curated_table
          AND sc.watermark_end   >= '${watermark_start}'
          AND sc.watermark_start <= '${watermark_end}')")

  if [ ${missing_inputs} -gt 0 ]; then
    audit_partial_source "${RUN_ID}" "${BATCH_ID}" "${gold_table}" \
      "Missing ${missing_inputs} Curated input(s) for ${gold_table}"
    fnLogMsg WARN "Skipping ${gold_table}: ${missing_inputs} Curated input(s) not COMPLETED"
    continue
  fi
```

Skip the target, name what is missing, let the run continue with the others.
**Partial-but-honest beats complete-but-silently-stale.**

> Note the third argument: this gate fires **before** any `source_control` rows are opened,
> so there is no Curated source row to mark PARTIAL. The `gold_table` goes in `source_name`
> instead, meaning "this target was skipped". It is the one place in the audit tables where
> `source_name` holds a Gold table rather than a Curated one — record that in
> `docs/GOLD_ANALYSIS.md` so nobody reading the table later mistakes it for corruption.
> G18's gate still catches it, because it counts any latest source row that is not COMPLETED.

### One source_control row per mapped input

All inputs share the target's `BATCH_ID`:

```bash
  for curated_table in $(audit_gold_sources_for "${gold_table}"); do
    input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${curated_table} WHERE ${watermark_condition}")

    audit_start_source "GOLD" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${curated_table}" \
      "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
  done

  # Driver input count drives stage_summary — enrichment sources add columns, not rows
  driver_table=$(audit_gold_driver_for "${gold_table}")
  input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${driver_table} WHERE ${watermark_condition}")
```

ENRICH and LOOKUP inputs still get a `source_control` row and a lineage edge — they are just
not the basis for counts. Their correctness is audited as RI rules in G16.

## After the write

```bash
  stage_start=$(date -u +"%Y-%m-%d %H:%M:%S")

  # ========== EXISTING SCD/MERGE PROCESSING — UNCHANGED ==========

  stage_end=$(date -u +"%Y-%m-%d %H:%M:%S")

  actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_gold}.${gold_table} WHERE _audit_batch_id='${BATCH_ID}'")
  # or, if the table has no audit columns, count the partition just written

  # One stage_summary per batch. source_name is the DRIVER — the full set of contributing
  # sources lives in audit_source_control and audit_lineage for this batch.
  audit_write_stage_summary "GOLD" "${RUN_ID}" "${BATCH_ID}" "${driver_table}" \
    "SCD2" "${gold_table}" "${input_count}" "${expected_count}" "${actual_count}" \
    "$((expected_count - actual_count))" "0" \
    "$([ ${expected_count} -eq ${actual_count} ] && echo 'MATCHED' || echo 'MISMATCHED')" \
    "COMPLETED" "${stage_start}" "${stage_end}" "" ""

  # Close every source row opened for this batch — not just the driver.
  # source_name is the 3rd argument: N rows share this batch_id.
  for curated_table in $(audit_gold_sources_for "${gold_table}"); do
    audit_complete_source "${RUN_ID}" "${BATCH_ID}" "${curated_table}" \
      "${input_count}" "${actual_count}" "0" "1" "0"
  done

done   # end gold_table loop
```

## Schema change: audit columns on Gold tables

If G14 confirmed schema changes are allowed:

1. Add `_audit_run_id STRING` and `_audit_batch_id STRING` to the Gold DDL.
2. Populate them in the SCD/MERGE HQL:

```sql
INSERT INTO ${gold_table}
SELECT ..., '${run_id}' AS _audit_run_id, '${batch_id}' AS _audit_batch_id
FROM ...
```

Pass via `--hivevar run_id=${RUN_ID} --hivevar batch_id=${BATCH_ID}`.

For SCD2 version closure, the existing row's audit columns stay unchanged — they record when
that version was created.

Note any table that cannot accept the columns.

## At pipeline end

A "source" here is a **(batch_id, source_name) pair**. With fan-in one target contributes
several, and because the model is append-only, count the **latest row per pair**:

```bash
latest_source_status="
  SELECT batch_id, source_name, status FROM (
    SELECT batch_id, source_name, status,
           ROW_NUMBER() OVER (PARTITION BY batch_id, source_name ORDER BY created_ts DESC) rn
    FROM ${AUDIT_DB}.audit_source_control
    WHERE layer='GOLD' AND run_id='${RUN_ID}') t
  WHERE rn = 1"

total_sources=$(audit_get_count     "SELECT COUNT(*) FROM (${latest_source_status}) s")
completed_sources=$(audit_get_count "SELECT COUNT(*) FROM (${latest_source_status}) s WHERE s.status='COMPLETED'")
failed_sources=$(audit_get_count    "SELECT COUNT(*) FROM (${latest_source_status}) s WHERE s.status='FAILED'")
partial_sources=$(audit_get_count   "SELECT COUNT(*) FROM (${latest_source_status}) s WHERE s.status='PARTIAL'")

if [ $((failed_sources + partial_sources)) -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" \
    "$((failed_sources + partial_sources))" \
    "failed:${failed_sources};partial_missing_inputs:${partial_sources}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "0"
fi
```

A target skipped by the fan-in gate is PARTIAL, not FAILED — nothing broke — but it must not
let the run finish COMPLETED. **This is the minimal end-of-run check; G18 replaces it** with
the full completion gate.

## Deliverables

1. Modified Gold master script with audit calls at run boundaries.
2. Target-scoped table loop: one `BATCH_ID` per Gold table, inner loop writing one
   `source_control` row per mapped Curated input.
3. Fan-in readiness gate, with skipped targets recorded PARTIAL and missing inputs named.
4. HQL modifications populating `_audit_run_id` / `_audit_batch_id`.
5. Modified sections shown as diffs with one-line safety justifications.
6. Any table that cannot accept the audit columns.

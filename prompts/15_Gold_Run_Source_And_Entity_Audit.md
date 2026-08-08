# Prompt 15 — Instrument Gold shell scripts: run, source, and entity audit

Add run/source/stage audit to the Gold pipeline shell scripts using the same audit
library (prompt 03b). Layer='GOLD'. Kill switch applies.

Do not change HiveQL transformations or business logic.

**Read `docs/GOLD_FANOUT_DESIGN.md` first.** Curated→Gold is many-to-many, so the batch
grain here is different from Curated's: **one batch per Gold target table, not per source
read.** A Gold table assembled from five Curated inputs produces five
`audit_source_control` rows sharing one `BATCH_ID`, and one `audit_stage_summary` row.
That is what keeps `_audit_batch_id` on the Gold row single-valued.

## At pipeline start

```bash
source ${AUDIT_LIB_PATH}/audit_functions.sh
audit_init || { fnLogMsg ERROR "Audit init failed"; }

RUN_ID=$(audit_generate_run_id)
audit_start_run "GOLD" "${RUN_ID}" "${PIPELINE_NAME}" "${PROCESS_NAME}" "${SOURCE_SYSTEM}"
```

## Per Gold target table — batch id, fan-in gate, then one source row per input

Iterate Gold targets in **dependency order** (`audit_gold_source_map.depends_on`), and
generate the batch id from the *target*:

```bash
for gold_table in $(audit_gold_targets_in_dependency_order); do
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${gold_table}")

  watermark_start="${CURATED_WATERMARK_START}"
  watermark_end="${CURATED_WATERMARK_END}"
```

### Fan-in readiness gate (before any build)

If a Gold table needs five Curated inputs and only four completed, the build still
produces a plausible row count and is quietly wrong — stale enrichment with no count
mismatch to catch it. Check every mapped input before building:

```bash
  missing_inputs=$(audit_get_count "
    SELECT COUNT(*) FROM ${AUDIT_DB}.audit_gold_source_map m
    WHERE m.gold_table = '${gold_table}' AND m.is_active = 'Y'
      AND NOT EXISTS (
        SELECT 1 FROM ${AUDIT_DB}.audit_source_control sc
        WHERE sc.layer = 'CURATED' AND sc.status = 'COMPLETED'
          AND sc.source_name = m.curated_table
          AND sc.watermark_end   >= '${watermark_start}'
          AND sc.watermark_start <= '${watermark_end}')")

  if [ ${missing_inputs} -gt 0 ]; then
    audit_partial_source "${RUN_ID}" "${BATCH_ID}" \
      "Missing ${missing_inputs} Curated input(s) for ${gold_table}"
    fnLogMsg WARN "Skipping ${gold_table}: ${missing_inputs} Curated input(s) not COMPLETED"
    continue
  fi
```

Skip the target, name what is missing, and let the run continue with other targets rather
than failing wholesale. Partial-but-honest beats complete-but-silently-stale.

### One source_control row per mapped Curated input

All inputs share the target's `BATCH_ID`:

```bash
  for curated_table in $(audit_gold_sources_for "${gold_table}"); do
    input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${curated_table} WHERE ${watermark_condition}")

    audit_start_source "GOLD" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${curated_table}" \
      "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
  done

  # driver input count — used as stage_summary input_count below
  driver_table=$(audit_gold_driver_for "${gold_table}")
  input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${driver_table} WHERE ${watermark_condition}")
```

ENRICH and LOOKUP inputs still get their own `source_control` row and lineage edge — they
are just not the basis for counts. Their correctness is audited as RI rules in prompt 16.

## Per Gold target table — after the write

After each Gold write (SCD apply, MERGE, etc.). `input_count` here is the **driver**
input count, since enrichment sources do not determine row population:
```bash
# Count written rows
# Option 1: If Gold has _audit_batch_id column
actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${gold_table} WHERE _audit_batch_id='${BATCH_ID}'")

# Option 2: If counting by partition
actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${gold_table} WHERE load_date='${current_date}'")

# One stage_summary row per batch. source_name is the DRIVER table — the full set of
# contributing sources lives in audit_source_control and audit_lineage for this batch.
audit_write_stage_summary "GOLD" "${RUN_ID}" "${BATCH_ID}" "${driver_table}" \
  "SCD2" "${gold_table}" "${input_count}" "${expected_count}" "${actual_count}" \
  "$((expected_count - actual_count))" "0" \
  "$([ ${expected_count} -eq ${actual_count} ] && echo 'MATCHED' || echo 'MISMATCHED')" \
  "COMPLETED" "${stage_start}" "${stage_end}" "" ""

# Close every source row opened for this batch — not just the driver
for curated_table in $(audit_gold_sources_for "${gold_table}"); do
  audit_complete_source "${RUN_ID}" "${BATCH_ID}" "${curated_table}"
done

done   # end gold_table loop
```

## Schema change: add audit columns to Gold tables

If prompt 14 confirmed schema changes are allowed:

1. Modify Gold DDL to add:
   - `_audit_run_id STRING`
   - `_audit_batch_id STRING`

2. Modify the SCD/MERGE HQL to populate them:
```sql
-- In INSERT or MERGE ... THEN INSERT
INSERT INTO ${gold_table}
SELECT 
  ...,
  '${run_id}' AS _audit_run_id,
  '${batch_id}' AS _audit_batch_id
FROM ...
```

Pass via `--hivevar run_id=${RUN_ID} --hivevar batch_id=${BATCH_ID}`.

For SCD2 version closure (UPDATE to set expiry date), the existing row's audit columns
remain unchanged — they record when that version was created.

Note any table that cannot accept the columns.

## At pipeline end

A "source" here is a **(batch_id, source_name) pair** — one Curated input into one Gold
target. With fan-in, one Gold target contributes several. And because the model is
append-only, count the **latest row per pair**, never all rows:

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

A target skipped by the fan-in gate is PARTIAL, not FAILED — nothing broke, an input was
missing — but it must not let the run finish COMPLETED. This is the minimal end-of-run
check; **prompt 18 replaces it** with the full completion gate.

## Deliverables

1. Modified Gold master script with audit calls at run boundaries
2. Target-scoped table loop: one `BATCH_ID` per Gold table, inner loop writing one
   `source_control` row per mapped Curated input
3. Fan-in readiness gate, with skipped targets recorded as PARTIAL and their missing
   inputs named
4. The `audit_gold_sources_for` / `audit_gold_driver_for` /
   `audit_gold_targets_in_dependency_order` helpers reading `audit_gold_source_map`
   (add to the shared audit library from prompt 03b)
5. HQL modifications to populate `_audit_run_id`, `_audit_batch_id`
6. Show modified sections as diffs with one-line safety justifications
7. Note any tables that cannot accept audit columns

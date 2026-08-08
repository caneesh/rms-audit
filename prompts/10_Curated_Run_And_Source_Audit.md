# Prompt 10 — Instrument Curated shell scripts: run and source audit

Add run-level and source-level audit to the Curated pipeline shell scripts using the
audit shell library (prompt 03b). Layer='CURATED'. Kill switch applies.

Do not change HiveQL transformations or business logic. If AUDIT_ENABLED=false, behavior
must be identical to today's.

## At pipeline start (master script)

```bash
source ${AUDIT_LIB_PATH}/audit_functions.sh
audit_init || { fnLogMsg ERROR "Audit init failed"; }  # continue even if audit fails

RUN_ID=$(audit_generate_run_id)
audit_start_run "CURATED" "${RUN_ID}" "${PIPELINE_NAME}" "${PROCESS_NAME}" "${SOURCE_SYSTEM}"
```

Note: There is no `spark_application_id` in shell. Leave that column NULL or capture the
YARN application ID if beeline exposes it (check `hivebeeline` output for app ID).

## Per source table processed

Before processing each source (inside the table loop):
```bash
BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${table_name}")
input_count=$(audit_get_count "SELECT COUNT(*) FROM ${source_table} WHERE ${watermark_condition}")
audit_start_source "CURATED" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${source_table}" \
  "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
```

Watermark values come from the trigger file or partition parameters found in prompt 09.

### When the Raw input is not ready

Prompt 09 item 8 asks how Raw dependencies are detected. If that check says the Raw side
has not completed for this window, do **not** process the table and do **not** mark it
FAILED — nothing broke. Record it as PARTIAL with the reason, and let the run continue
with the other tables:

```bash
audit_partial_source "${RUN_ID}" "${BATCH_ID}" "${source_table}" \
  "Raw input not COMPLETED for window ${watermark_start}..${watermark_end}"
continue
```

PARTIAL must block a COMPLETED run — the completion gate in prompt 13 counts any latest
source row that is not COMPLETED. This is the Curated equivalent of Gold's fan-in gate,
and it exists for the same reason: a table built from an input that never arrived is
silently stale rather than obviously wrong.

## Per target table written

After each MERGE or INSERT completes:
```bash
actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${target_table} WHERE _audit_batch_id='${BATCH_ID}'")
# OR if no audit columns yet: count the partition just written
audit_write_stage_summary "CURATED" "${RUN_ID}" "${BATCH_ID}" "${source_table}" \
  "MERGE" "${target_table}" "${input_count}" "${expected_count}" "${actual_count}" ...
```

## Lineage

For each Curated target, record which Raw batches contributed. Query the Raw layer's
`audit_source_control` to find batch_ids that overlap the watermark window:
Select `run_id` alongside `batch_id` — `audit_write_lineage` needs both, and a batch id on
its own does not identify the run that produced it:

```bash
# Find Raw batches that fed this Curated batch
raw_batches_query="
  SELECT DISTINCT run_id, batch_id
  FROM ${AUDIT_DB}.audit_source_control 
  WHERE layer='RAW' 
    AND source_name='${raw_table}'
    AND status='COMPLETED'
    AND watermark_end >= '${curated_watermark_start}'
    AND watermark_start <= '${curated_watermark_end}'
"
while IFS=$'\t' read -r raw_run_id raw_batch; do
  [ -z "${raw_batch}" ] && continue
  audit_write_lineage "CURATED" "${RUN_ID}" "${BATCH_ID}" "${target_table}" \
    "RAW" "${raw_run_id}" "${raw_batch}" "${raw_table}"
done < <(hivebeeline --silent=true --outputformat=tsv2 -e "${raw_batches_query}" 2>/dev/null | grep -v "^$")
```

If the lookup is ambiguous or too expensive, record source_name + window and leave
source_batch_id NULL — do not guess.

## Schema change: add audit columns to Curated tables

If prompt 09 confirmed schema changes are allowed, modify the Curated DDL to add:
- `_audit_run_id STRING`
- `_audit_batch_id STRING`

And modify the MERGE/INSERT HQL to populate them:
```sql
-- In the MERGE or INSERT ... SELECT
SELECT 
  ...,
  '${run_id}' AS _audit_run_id,
  '${batch_id}' AS _audit_batch_id
FROM ...
```

Pass via `--hivevar run_id=${RUN_ID} --hivevar batch_id=${BATCH_ID}`.

If a table cannot accept the columns, note it and audit at batch level only.

## At pipeline end

Count the **latest row per source**, not all rows. The model is append-only, so a source
that went STARTED → FAILED → COMPLETED on retry has three rows: mixing
`COUNT(DISTINCT source_name)` for the total with a bare `COUNT(*)` for the others can
report more completed sources than exist, and counts a recovered source as both completed
and failed.

```bash
# Aggregate totals from this run — one row per source, latest status wins
latest_source_status="
  SELECT batch_id, source_name, status FROM (
    SELECT batch_id, source_name, status,
           ROW_NUMBER() OVER (PARTITION BY batch_id, source_name ORDER BY created_ts DESC) rn
    FROM ${AUDIT_DB}.audit_source_control
    WHERE layer='CURATED' AND run_id='${RUN_ID}') t
  WHERE rn = 1"

total_sources=$(audit_get_count     "SELECT COUNT(*) FROM (${latest_source_status}) s")
completed_sources=$(audit_get_count "SELECT COUNT(*) FROM (${latest_source_status}) s WHERE s.status='COMPLETED'")
failed_sources=$(audit_get_count    "SELECT COUNT(*) FROM (${latest_source_status}) s WHERE s.status='FAILED'")

if [ ${failed_sources} -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "${failed_sources}" "Some sources failed"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "0"
fi
```

## Deliverables

1. Modified master shell script with audit calls at run boundaries
2. Modified table-loop section with source/stage audit calls
3. HQL modifications to populate `_audit_run_id`, `_audit_batch_id`
4. Lineage capture logic
5. Show modified sections as diffs with one-line safety justifications

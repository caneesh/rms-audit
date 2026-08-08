# R05 — Fix the Curated instrumentation

**Closes the R02–R05 atomic block. After this, run R06 before anything else.**

## Baseline

Patches the generated Curated instrumentation from `prompts/10`, `prompts/11` and
`prompts/13`. Use `docs/AUDIT_V2_BASELINE.md` (R01) for paths, and R04's call-site
enumeration as the checklist.

`prompts/12_Curated_CDC_Merge_Audit.md` is **not** affected — its counts aggregate CDC
staging data, not audit rows, which is correct.

Curated does **not** need the Gold fan-in design. Raw→Curated is one-to-one at table level
(confirmed against `reference/hive_raw_curated_mapping.xlsx`), so there is no source map, no
driver roles, and no per-edge grain here. The `source_name` / `target_table` columns still get
populated, because the table is shared with Gold where they carry the identity.

If R01 found that any Curated table reads **two or more** Raw tables, stop — that table has
Gold's fan-in problem and `docs/GOLD_FANOUT_DESIGN.md` applies to it. Say so rather than
quietly summing the inputs.

---

## Fix 1 — the double-count bug (do this first)

**This is a live defect, not a hardening change.**

The reconciliation logic computes:

```bash
source_count=$(audit_get_count "
  SELECT SUM(input_count) FROM ${AUDIT_DB}.audit_source_control
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'")
```

`audit_source_control` is append-only. `audit_start_source` writes `input_count`, and
`audit_complete_source` writes it **again**. So this `SUM` returns **twice** the real source
count, every reconciliation reports a huge unexplained difference, and MISMATCHED fails the
run. Replace with the latest row, scoped to layer, batch and source:

```bash
source_count=$(audit_get_count "
  SELECT input_count
  FROM ${AUDIT_DB}.audit_source_control
  WHERE layer='CURATED' AND run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
    AND source_name='${source_table}'
  ORDER BY created_ts DESC LIMIT 1")
```

The same applies to `target_count` from `audit_stage_summary`, which in `prompts/13` has no
`ORDER BY`/`LIMIT` and so returns multiple rows after a retry:

```bash
target_count=$(audit_get_count "
  SELECT actual_written_count
  FROM ${AUDIT_DB}.audit_stage_summary
  WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' AND target_table='${target_table}'
  ORDER BY created_ts DESC LIMIT 1")
```

**Search the whole workspace for any other aggregate over an audit table** — `SUM(`,
`COUNT(*)`, `MAX(` against `audit_source_control`, `audit_stage_summary`,
`audit_merge_summary` or `audit_reconciliation`. Each one is suspect. The rule: reduce to the
latest row per key first, then aggregate across keys if you need to.

## Fix 2 — reconciliation HQL is missing the new columns

If reconciliation was implemented as an HQL INSERT (R01 section 1 says which), it selects a
fixed column list and will now **fail on column count**. Add both, in R02's position:

```sql
  'CURATED' AS layer,
  '${run_id}' AS run_id,
  '${entity_name}' AS entity_name,
  'RAW' AS from_layer,
  'CURATED' AS to_layer,
  '${source_table}' AS source_name,      -- NEW
  '${target_table}' AS target_table,     -- NEW
  ${source_count} AS source_count,
  ...
```

## Fix 3 — every changed call site

Work through R04's enumeration. Three functions changed shape:

| Function | New argument order |
|---|---|
| `audit_complete_source` | `run_id, batch_id, source_name, input_count, processed_count, rejected_count, completed_targets, failed_targets` |
| `audit_fail_source` | `run_id, batch_id, source_name, error_message` |
| `audit_write_reconciliation` | `layer, run_id, entity_name, from_layer, to_layer, source_name, target_table, source_count, target_count, rejected_count, filtered_count, difference, expected_reason, unexplained, status` |

Known call sites from the original prompts — expect more if the instrumentation was
copy-pasted across scripts:

- `prompts/10` — `audit_complete_source` at the end of the table loop
- `prompts/11` — `audit_fail_source` on blocking rule failure
- `prompts/13` — `audit_write_reconciliation`

**Grep for the function names; do not rely on this list.** A missed call site produces rows
with values silently shifted one column left, which no test will catch and which looks like
plausible data.

## Fix 4 — `${raw_run_id}` is never defined

In the lineage loop (`prompts/10`), the query selects only `batch_id` but the write passes
`${raw_run_id}`. Every Curated lineage edge carries an empty source run id. Select both:

```bash
raw_batches_query="
  SELECT DISTINCT run_id, batch_id
  FROM ${AUDIT_DB}.audit_source_control
  WHERE layer='RAW' AND source_name='${raw_table}' AND status='COMPLETED'
    AND watermark_end   >= '${curated_watermark_start}'
    AND watermark_start <= '${curated_watermark_end}'"

while IFS=$'\t' read -r raw_run_id raw_batch; do
  [ -z "${raw_batch}" ] && continue
  audit_write_lineage "CURATED" "${RUN_ID}" "${BATCH_ID}" "${target_table}" \
    "RAW" "${raw_run_id}" "${raw_batch}" "${raw_table}"
done < <(hivebeeline --silent=true --outputformat=tsv2 -e "${raw_batches_query}" 2>/dev/null | grep -v "^$")
```

## Fix 5 — end-of-run tallies

`prompts/10` mixes `COUNT(DISTINCT source_name)` for the total with a bare `COUNT(*)` for
completed and failed. Under append-only, a source that went STARTED → FAILED → COMPLETED on
retry is counted as both completed and failed, and the totals disagree with each other.

```bash
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
partial_sources=$(audit_get_count   "SELECT COUNT(*) FROM (${latest_source_status}) s WHERE s.status='PARTIAL'")
```

A PARTIAL source must block a COMPLETED run without being reported as a failure:

```bash
if [ $((failed_sources + partial_sources)) -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" \
    "$((failed_sources + partial_sources))" \
    "failed:${failed_sources};partial:${partial_sources}"
  exit 1
fi
```

## Fix 6 — completion gate needs a layer filter

The five gate queries in `prompts/13` hit the shared audit tables without `layer='CURATED'`.
Run ids are unique in practice, but an unfiltered gate silently widens as soon as another
layer writes under the same id — which is exactly what happens once Gold starts. Add the
filter to all five, and make the `sources_failed` check latest-row-per-source using the
`latest_source_status` pattern from Fix 5, counting anything not COMPLETED.

## Fix 7 — rule-name placeholder is inconsistent

`prompts/13` uses `${FILTER_RULES}`; the Gold prompts and the config use
`${FILTERING_RULES}`. Standardise on **`FILTERING_RULES`** and update the config key in
`audit_config.sh` to match. Confirm the three rule lists are all defined and distinct:
`BLOCKING_RULES` (stop the load), `REJECTING_RULES` (rows rejected), `FILTERING_RULES`
(intentional drops that explain a difference).

## Fix 8 — new PARTIAL path when the Raw input is not ready

`prompts/09` item 8 asks how Raw dependencies are detected, but there was nowhere for the
answer to go. If the Raw side has not completed for this window, do not process the table and
do not mark it FAILED — nothing broke:

```bash
audit_partial_source "${RUN_ID}" "${BATCH_ID}" "${source_table}" \
  "Raw input not COMPLETED for window ${watermark_start}..${watermark_end}"
continue
```

Same reasoning as Gold's fan-in gate: a table built from an input that never arrived is
silently stale rather than obviously wrong.

---

## What not to change

- No business logic, no HiveQL transformations, no Curated table schemas beyond the two
  audit columns that already exist.
- **No Hive `UPDATE`.** The append-only rule is exactly what a well-meaning fix to "duplicate
  rows in audit_source_control" would break. Those duplicates are correct — they are the
  status history.
- Do not "fix" the double-count by removing the second write in `audit_complete_source`. The
  second row is the append-only status transition and must stay; the reader is what was wrong.

## Deliverables

1. Each fix shown as a diff, with a one-line safety justification.
2. The complete list of call sites found and updated, cross-checked against R04's enumeration.
3. Any other audit-table aggregate found by the Fix 1 sweep, with what you did about it.
4. Explicit confirmation that no `UPDATE` statement was introduced.
5. Anything in the workspace that contradicts these instructions — say so rather than forcing
   the change through.

## Next

R06 verifies all of this against real data. Do not skip it — Fix 1 and Fix 3 are both the
kind of defect that only shows up at runtime.

# R04 — Fix the shell audit library

**Part of the R02–R05 atomic block. Do not run any pipeline until R05 is complete.**

## Baseline

Patches the generated `audit_functions.sh` from `prompts/03b_Shell_Audit_Library.md`. Use
`docs/AUDIT_V2_BASELINE.md` (R01) section 2 for the **actual** signatures as generated — if
they already differ from what `prompts/03b` described, adapt these instructions to what is
really there and say what you changed.

All existing rules from `prompts/03b` still hold: kill switch checked at function entry,
audit failures never mask business failures, UTC timestamps, single-quote escaping, and the
config keys in `audit_init`.

## Change 1 — `source_name` becomes a parameter on three functions

### Why

Gold fan-in means one batch has **N open source rows** — one per Curated table feeding that
Gold target. `batch_id` alone no longer identifies which source row to close. Curated is
one-to-one and does not strictly need this, but the library is shared, so the signature must
support the Gold case.

### What

`source_name` is inserted as the **third** parameter, after `run_id` and `batch_id`,
shifting everything after it right by one.

```bash
audit_complete_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  local run_id="$1"
  local batch_id="$2"
  local source_name="$3"          # NEW — 3rd position
  local input_count="$4"
  local processed_count="$5"
  local rejected_count="$6"
  local completed_targets="$7"
  local failed_targets="$8"
  # INSERT new row with status=COMPLETED
}

audit_fail_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  local run_id="$1"
  local batch_id="$2"
  local source_name="$3"          # NEW — 3rd position
  local error_message="$4"
  # INSERT new row with status=FAILED
}
```

> **This is the highest-risk change in the whole v2 set.** Bash arguments are positional and
> untyped. An un-updated caller passes `input_count` where `source_name` is now expected —
> nothing errors, and every row lands with values in the wrong columns. R05 must find every
> caller. Do not consider R04 done until R05 is done.

## Change 2 — new `audit_partial_source`

PARTIAL already exists in the status vocabulary (`prompts/02`) and on the Scala side
(`markSourcePartial`), but the shell library never got it.

```bash
audit_partial_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  local run_id="$1"
  local batch_id="$2"
  local source_name="$3"
  local reason="$4"     # e.g. "Missing Curated inputs: curated_plan,curated_addr"
  # INSERT new row with status=PARTIAL and error_message=reason
}
```

PARTIAL means *nothing broke, but this unit was not fully processed* — an input that was not
ready. It must **not** be reported as a failure, and it must **block** a COMPLETED run.

## Change 3 — `audit_write_reconciliation` gains two parameters

Match the R02 column order exactly — this function inserts positionally.

```bash
audit_write_reconciliation() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  local layer="$1"
  local run_id="$2"
  local entity_name="$3"
  local from_layer="$4"
  local to_layer="$5"
  local source_name="$6"      # NEW — source table for this edge
  local target_table="$7"     # NEW — target table for this edge
  local source_count="$8"
  local target_count="$9"
  local rejected_count="${10}"
  local filtered_count="${11}"
  local difference="${12}"
  local expected_reason="${13}"
  local unexplained="${14}"
  local status="${15}"
  # INSERT INTO ${AUDIT_DB}.audit_reconciliation VALUES (...)
}
```

Add a comment above it: one row per **edge**, and `source_count` / `target_count` are
**distinct natural keys, not physical rows**.

## Change 4 — `audit_get_value`

`audit_get_count` filters output to numerics. The Gold map helpers return table names, so
they need a scalar helper that does not.

```bash
audit_get_value() {
  local query="$1"
  ${AUDIT_BEELINE_CMD} --silent=true --outputformat=tsv2 -e "$query" 2>/dev/null | tail -1
}
```

## Change 5 — Gold source-map helpers

These read `audit_gold_source_map` (created in R02, populated by G14). They exist so that
Gold instrumentation never hardcodes the Curated→Gold mapping. Each returns
whitespace-separated values on stdout, empty on no match.

```bash
audit_gold_sources_for() {
  # ALL active Curated inputs for a Gold table — DRIVER, ENRICH and LOOKUP alike.
  # Every one gets a source_control row and a lineage edge.
  local gold_table="$1"
  audit_get_value "SELECT CONCAT_WS(' ', COLLECT_LIST(curated_table))
                   FROM ${AUDIT_DB}.audit_gold_source_map
                   WHERE gold_table='${gold_table}' AND is_active='Y'"
}

audit_gold_driver_for() {
  # The single DRIVER input — the one whose row population determines the Gold table's.
  # Only this one gets a reconciliation identity in G18.
  local gold_table="$1"
  audit_get_value "SELECT curated_table FROM ${AUDIT_DB}.audit_gold_source_map
                   WHERE gold_table='${gold_table}' AND is_active='Y'
                     AND source_role='DRIVER' LIMIT 1"
}

audit_gold_targets_in_dependency_order() {
  # Gold tables ordered so anything named in depends_on is built first.
  # Assumes a one-level graph (a fact table depending on its dimensions).
  # If G14 finds deeper nesting, replace this with a real topological sort and say so.
  audit_get_value "SELECT CONCAT_WS(' ', COLLECT_LIST(gold_table)) FROM (
                     SELECT DISTINCT gold_table,
                            CASE WHEN depends_on IS NULL OR depends_on='' THEN 0 ELSE 1 END AS lvl
                     FROM ${AUDIT_DB}.audit_gold_source_map WHERE is_active='Y'
                     ORDER BY lvl, gold_table) t"
}
```

**Cache the map.** These run per Gold table inside the main loop, and a beeline round trip
each is wasteful. Read `audit_gold_source_map` once in `audit_init` into shell variables or a
temp file and have the helpers read that. A stale cache *within* a run is fine — it is
reference data, not events — but it must be refreshed per run. If you implement caching, say
so in the output.

## Deliverables

1. Updated `audit_functions.sh`, with each changed or added function shown as a diff.
2. Updated function inventory documentation.
3. Confirmation that the kill switch guard is present on every new function.
4. A list of every file in the workspace that calls the three changed functions — this is
   the input to R05. Do not fix them here; just enumerate them so nothing is missed.

## Do not stop here

R05 must follow immediately. Between this prompt and R05 the Curated callers are broken.

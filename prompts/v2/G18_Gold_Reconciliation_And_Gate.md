# G18 — Curated → Gold reconciliation and the completion gate

## Baseline

Supersedes `prompts/18_End_To_End_Reconciliation.md`, which reconciled at layer grain. That
has no checkable identity under many-to-many.

## Inputs — read these before writing anything

- `docs/GOLD_ANALYSIS.md` (G14) — the **per-edge expected relationship sentences** (these
  become `expected_difference_reason`), the **natural key per entity** (counts here are
  distinct keys, not rows), and `${REJECTING_RULES}` / `${FILTERING_RULES}`.
- `${AUDIT_DB}.audit_gold_source_map` — drives the DRIVER-edge loop and the lineage gate.
- `docs/GOLD_FANOUT_DESIGN.md` §5.

The reconciliation identity is the one thing in this pack that **cannot be derived from the
code** — it is a business statement about what the Gold table is supposed to contain. If G14
did not produce a sentence for an edge, ask for it. **Do not infer one from observed counts:**
that makes the check tautological, so it passes on day one and never catches a regression.

## Three ways this differs from Curated

1. **Per edge, not per entity.** `curated_member → GOLD` is not checkable — that table may
   feed three Gold tables under three different rules. Write one row per (source table →
   target table), using the `source_name` / `target_table` columns added in R02.
2. **DRIVER edges only.** ENRICH and LOOKUP inputs add columns, not rows. They were audited
   as RI rules in G16.
3. **Distinct natural keys, not rows.** SCD2 inflates the target side with version rows;
   fan-in enrichment inflates the source side. Keys are stable under both.

| edge | expected relationship |
|---|---|
| curated_member → gold_member_dim | 1:1 on member key |
| curated_member → gold_member_coverage | only members with active coverage; filter reason recorded |
| curated_member → gold_member_addr_hist | SCD2; version rows expected to exceed keys |

*(illustrative — use the real edges from G14)*

**Never sum across edges.** A Curated table's count does not equal the total across the Gold
targets it feeds. Each edge is an independent identity, evaluated alone.

## The identity

```
distinct_source_keys_in_scope
    = distinct_target_keys_touched + rejected_keys + filtered_keys
```

## Reconciliation

All counts come from audit rows already written this run — **no new table scans.**

```bash
for gold_table in $(audit_gold_targets_in_dependency_order); do
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${gold_table}")
  driver_table=$(audit_gold_driver_for "${gold_table}")

  # Source: Curated keys in scope for THIS edge.
  # MUST be scoped to BATCH_ID and take the LATEST row. Under fan-out the same source_name
  # appears under several Gold batches, so summing across the run multiplies by the fan-out;
  # and the model is append-only, so SUM over a single batch double-counts.
  source_count=$(audit_get_count "
    SELECT input_count FROM ${AUDIT_DB}.audit_source_control
    WHERE layer='GOLD' AND run_id='${RUN_ID}'
      AND batch_id='${BATCH_ID}' AND source_name='${driver_table}'
    ORDER BY created_ts DESC LIMIT 1")

  # Target: Gold keys TOUCHED, not rows written. inserted+updated counts key operations,
  # so SCD2 version rows do not inflate it.
  target_count=$(audit_get_count "
    SELECT COALESCE(SUM(inserted_count + updated_count), 0)
    FROM ${AUDIT_DB}.audit_merge_summary
    WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}' AND target_table='${gold_table}'")

  # Rejected: blocking rule failures
  rejected_count=$(audit_get_count "
    SELECT COALESCE(SUM(failed_count), 0) FROM ${AUDIT_DB}.audit_rule_result
    WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
      AND status='FAILED' AND rule_name IN (${REJECTING_RULES})")

  # Filtered: intentional drops, each with a named reason
  filtered_count=0
  expected_reason=""
  while IFS=$'\t' read -r rule_name count; do
    [ -z "$count" ] && continue
    filtered_count=$((filtered_count + count))
    expected_reason="${expected_reason}${rule_name}:${count};"
  done < <(${AUDIT_BEELINE_CMD} --silent=true -e "
    SELECT rule_name, failed_count FROM ${AUDIT_DB}.audit_rule_result
    WHERE run_id='${RUN_ID}' AND batch_id='${BATCH_ID}'
      AND rule_type='BUSINESS' AND rule_name IN (${FILTERING_RULES})" 2>/dev/null | grep -v "^$")

  # Prepend this edge's expected relationship sentence from docs/GOLD_ANALYSIS.md
  expected_reason="${EDGE_RELATIONSHIP_FOR_THIS_EDGE};${expected_reason}"

  unexplained=$((source_count - target_count - rejected_count - filtered_count))

  if [ ${unexplained} -eq 0 ]; then
    [ ${filtered_count} -eq 0 ] && recon_status="MATCHED" || recon_status="EXPLAINED"
  else
    recon_status="MISMATCHED"
  fi

  audit_write_reconciliation "GOLD" "${RUN_ID}" "${entity_name}" \
    "CURATED" "GOLD" \
    "${driver_table}" "${gold_table}" \
    "${source_count}" "${target_count}" "${rejected_count}" "${filtered_count}" \
    "$((source_count - target_count))" "${expected_reason}" "${unexplained}" \
    "${recon_status}"

  if [ "${recon_status}" = "MISMATCHED" ]; then
    fnLogMsg ERROR "Reconciliation MISMATCHED ${driver_table} -> ${gold_table}: unexplained=${unexplained}"
  fi
done
```

If G16 reported that the SCD logic cannot produce per-row actions, `target_count` falls back
to `stage_summary.actual_written_count` — valid **only** when the SCD writes one row per key.
State which you used.

MISMATCHED fails the Gold run.

## Completion gate

Replaces the minimal end-of-run check in G15. COMPLETED only when **all** hold:

```bash
sources_incomplete=$(audit_get_count "
  SELECT COUNT(*) FROM (
    SELECT batch_id, source_name, status FROM (
      SELECT batch_id, source_name, status,
             ROW_NUMBER() OVER (PARTITION BY batch_id, source_name ORDER BY created_ts DESC) rn
      FROM ${AUDIT_DB}.audit_source_control
      WHERE layer='GOLD' AND run_id='${RUN_ID}') t
    WHERE rn=1 AND status <> 'COMPLETED') x")

blocking_rules_failed=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result
  WHERE layer='GOLD' AND run_id='${RUN_ID}'
    AND status='FAILED' AND rule_name IN (${BLOCKING_RULES})")

ri_checks_failed=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_rule_result
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND rule_type='RI' AND status='FAILED'")

merges_failed=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_merge_summary
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='FAILED'")

recon_mismatched=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_reconciliation
  WHERE layer='GOLD' AND run_id='${RUN_ID}' AND status='MISMATCHED'")

# Lineage COMPLETE, not merely present. "At least one edge exists" passes when 1 of 5
# sources was recorded — compare the edge count against the map instead.
lineage_incomplete=$(audit_get_count "
  SELECT COUNT(*) FROM (
    SELECT l.target_table
    FROM (SELECT target_table, COUNT(DISTINCT source_name) AS c
          FROM ${AUDIT_DB}.audit_lineage
          WHERE layer='GOLD' AND run_id='${RUN_ID}' GROUP BY target_table) l
    JOIN (SELECT gold_table, COUNT(DISTINCT curated_table) AS c
          FROM ${AUDIT_DB}.audit_gold_source_map
          WHERE is_active='Y' GROUP BY gold_table) m
      ON l.target_table = m.gold_table
    WHERE l.c <> m.c) x")

fatal_errors=$(audit_get_count "
  SELECT COUNT(*) FROM ${AUDIT_DB}.audit_error_detail
  WHERE layer='GOLD' AND run_id='${RUN_ID}'
    AND error_type IN ('FATAL','MERGE_FAILURE','SCD_FAILURE')")

error_reasons=""
[ ${sources_incomplete}    -gt 0 ] && error_reasons="${error_reasons}sources_incomplete:${sources_incomplete};"
[ ${blocking_rules_failed} -gt 0 ] && error_reasons="${error_reasons}blocking_rules_failed:${blocking_rules_failed};"
[ ${ri_checks_failed}      -gt 0 ] && error_reasons="${error_reasons}ri_failed:${ri_checks_failed};"
[ ${merges_failed}         -gt 0 ] && error_reasons="${error_reasons}merges_failed:${merges_failed};"
[ ${recon_mismatched}      -gt 0 ] && error_reasons="${error_reasons}recon_mismatched:${recon_mismatched};"
[ ${lineage_incomplete}    -gt 0 ] && error_reasons="${error_reasons}lineage_incomplete:${lineage_incomplete};"
[ ${fatal_errors}          -gt 0 ] && error_reasons="${error_reasons}fatal_errors:${fatal_errors};"

if [ -n "${error_reasons}" ]; then
  audit_fail_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "${sources_incomplete}" "${error_reasons}"
  fnLogMsg ERROR "Gold run FAILED: ${error_reasons}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${total_sources}" "${completed_sources}" "0"
  fnLogMsg INFO "Gold run COMPLETED successfully"
fi
```

`sources_incomplete` uses latest-row-per-(batch, source) because the model is append-only
and, under fan-in, one batch has several source rows. It also catches the PARTIAL rows G15's
fan-in gate writes for skipped targets.

## Verification

After the first successful Gold run, confirm:

- One `audit_reconciliation` row per DRIVER edge, each with non-null `source_name` and
  `target_table` holding **table names**, not numbers.
- `unexplained_difference = 0` on a clean batch.
- `audit_lineage` edge count per target equals the active source count in the map.
- A target skipped by the fan-in gate appears as PARTIAL and the run did **not** complete.

## Deliverables

1. Per-edge reconciliation, DRIVER edges only, counted in distinct natural keys, with the
   per-edge `expected_difference_reason` from G14.
2. The completion gate, including `lineage_incomplete`.
3. Integration into the Gold master script.
4. A statement of which `target_count` source you used and why.

## Not in scope

The `v_reconciliation_daily` view belongs to Phase 4 (`prompts/19`). When it is built it must
be at **edge grain** — one row per Curated→Gold edge per day, chained on
`rc.target_table = rg.source_name` rather than on `entity_name` — and its RAW rollup must
deduplicate to the latest row per `(batch_id, source_name)` before summing. See
`prompts/v2/00_README.md` §5.

# Prompt 18a — Add Gold reconciliation and the completion gate

Two things: per-edge Curated→Gold reconciliation, then the gate that decides whether the
run is COMPLETED.

**Context:**
- Rules, merge summary and lineage are written (15a, 16a, 17a)
- `${AUDIT_DB}.audit_gold_source_map` is loaded (from 14a)
- The per-edge expected relationships and natural keys come from `docs/GOLD_ANALYSIS.md`
- All counts come from audit rows already written this run — **no new table scans**

## Reconcile per edge, on distinct keys, DRIVER only

Three things differ from Curated (13a):

1. **Per edge, not per entity.** `curated_member → GOLD` is not checkable — that table may
   feed three Gold tables under three different rules. Write one row per (source table →
   target table), using the `source_name` / `target_table` columns.
2. **DRIVER edges only.** ENRICH and LOOKUP inputs add columns, not rows; counting them is
   meaningless. They were audited as RI rules in 16a.
3. **Distinct natural keys, not rows.** SCD2 inflates the target side with version rows,
   and fan-in enrichment inflates the source side. Keys are stable under both.

**Add after lineage, before closing the batch:**

```bash
# ============ RECONCILIATION (one row per DRIVER edge) ============
driver_table=$(audit_gold_driver_for "${gold_table}")

# Source: Curated keys in scope for THIS edge.
# MUST be scoped to BATCH_ID — under fan-out the same source_name appears under several
# Gold batches, and summing across the run multiplies the source count by the fan-out.
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

# Filtered: intentional drops, with a named reason each (same pattern as 13a)
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

# Prepend the edge's expected relationship from docs/GOLD_ANALYSIS.md, e.g.
#   "curated_member keys in window = gold_member_dim keys touched"
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
  FAILED_SOURCES=$((FAILED_SOURCES + 1))
fi
```

**Do not infer the expected relationship from observed counts.** If 14a did not give you a
sentence for an edge, ask for it. A reason derived from what the run actually produced is
tautological — it passes on day one and never catches a regression.

## Completion gate

Replace the simple end-of-run check from 15a. The run is COMPLETED only if **all** hold:

```bash
# ============ COMPLETION GATE ============
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
  audit_fail_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "${sources_incomplete}" "${error_reasons}"
  fnLogMsg ERROR "Gold run FAILED: ${error_reasons}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "0"
  fnLogMsg INFO "Gold run COMPLETED successfully"
fi
```

`sources_incomplete` uses latest-row-per-(batch, source) because the model is append-only
and, under fan-in, one batch has several source rows. It also catches the PARTIAL rows
that 15a's fan-in gate writes for skipped targets.

Next: **18b** builds the reporting view over these reconciliation rows.

[PASTE: The end of your Gold master script]

# Prompt 15a — Add Gold run/source audit

Add audit instrumentation to Gold scripts. Same pattern as Curated (10a, 10b).

**Context:**
- Shell audit functions exist (S02/S03)
- Gold uses similar patterns to Curated but with SCD logic
- layer = "GOLD"
- **Curated→Gold is many-to-many.** One Gold table is built from several Curated tables
  (fan-in) and one Curated table feeds several Gold tables (fan-out). So `BATCH_ID` is
  scoped to the **Gold target**, and each target gets N `source_control` rows sharing that
  batch id — one per mapped Curated input. See `docs/GOLD_FANOUT_DESIGN.md`.
- The mapping lives in `audit_gold_source_map` (gold_table, curated_table, source_role,
  depends_on). `source_role` is DRIVER / ENRICH / LOOKUP; only the DRIVER determines row
  counts.

**Add to Gold master script (same pattern as Curated 10a):**

```bash
#!/bin/bash
# Source audit
source /path/to/audit_config.sh
source /path/to/audit_functions.sh
audit_init

RUN_ID=$(audit_generate_run_id)
export RUN_ID

audit_start_run "GOLD" "${RUN_ID}" "GoldPipeline" "SCD_Load" "RMS"

TOTAL_SOURCES=0
COMPLETED_SOURCES=0
FAILED_SOURCES=0
PARTIAL_SOURCES=0   # targets skipped by the fan-in gate
```

**Add to table loop (source audit):**

```bash
# Iterate targets in dependency order, NOT the flat ${goldTablesFromParams} list
for target_table in $(audit_gold_targets_in_dependency_order); do
  BATCH_ID=$(audit_generate_batch_id "${RUN_ID}" "${target_table}")

  # --- Fan-in gate: every mapped Curated input must be COMPLETED for this window ---
  missing_inputs=$(audit_get_count "
    SELECT COUNT(*) FROM ${AUDIT_DB}.audit_gold_source_map m
    WHERE m.gold_table='${target_table}' AND m.is_active='Y'
      AND NOT EXISTS (
        SELECT 1 FROM ${AUDIT_DB}.audit_source_control sc
        WHERE sc.layer='CURATED' AND sc.status='COMPLETED'
          AND sc.source_name = m.curated_table
          AND sc.watermark_end   >= '${watermark_start}'
          AND sc.watermark_start <= '${watermark_end}')")

  if [ ${missing_inputs} -gt 0 ]; then
    audit_partial_source "${RUN_ID}" "${BATCH_ID}" "Missing ${missing_inputs} Curated input(s)"
    PARTIAL_SOURCES=$((PARTIAL_SOURCES + 1))
    continue
  fi

  # --- One source_control row per mapped Curated input, all sharing BATCH_ID ---
  for curated_source in $(audit_gold_sources_for "${target_table}"); do
    TOTAL_SOURCES=$((TOTAL_SOURCES + 1))
    input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${curated_source} WHERE ${watermark_condition}")

    audit_start_source "GOLD" "${RUN_ID}" "${BATCH_ID}" "TABLE" "${curated_source}" \
      "" "" "" "" "${watermark_start}" "${watermark_end}" "${input_count}"
  done

  # Driver input count drives the stage_summary — enrichment sources add columns, not rows
  driver_table=$(audit_gold_driver_for "${target_table}")
  input_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_curated}.${driver_table} WHERE ${watermark_condition}")

  stage_start=$(date -u +"%Y-%m-%d %H:%M:%S")
  
  # ========== EXISTING SCD/MERGE PROCESSING ==========
  # Your existing Gold processing code
  # ====================================================
  
  stage_end=$(date -u +"%Y-%m-%d %H:%M:%S")
  
  # On success: one stage_summary per BATCH (not per source)
  actual_count=$(audit_get_count "SELECT COUNT(*) FROM ${hiveDB_gold}.${target_table} WHERE _audit_batch_id='${BATCH_ID}'")
  
  audit_write_stage_summary "GOLD" "${RUN_ID}" "${BATCH_ID}" "${driver_table}" \
    "SCD2" "${target_table}" "${input_count}" "${input_count}" "${actual_count}" \
    "$((input_count - actual_count))" "0" "MATCHED" "COMPLETED" \
    "${stage_start}" "${stage_end}" ""
  
  # Close every source row opened for this batch, not just the driver
  for curated_source in $(audit_gold_sources_for "${target_table}"); do
    audit_complete_source "${RUN_ID}" "${BATCH_ID}" "${curated_source}"
    COMPLETED_SOURCES=$((COMPLETED_SOURCES + 1))
  done
done

# End of run
# A target skipped for missing inputs is PARTIAL, not FAILED — but it still blocks COMPLETED
if [ $((FAILED_SOURCES + PARTIAL_SOURCES)) -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" \
    "$((FAILED_SOURCES + PARTIAL_SOURCES))" \
    "failed:${FAILED_SOURCES};partial_missing_inputs:${PARTIAL_SOURCES}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "0"
fi
```

Show the diff of changes to your Gold scripts.

[PASTE: Your Gold master script or table loop]

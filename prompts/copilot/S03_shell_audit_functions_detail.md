# Prompt S03 — Create shell audit functions (stage/error)

Continue the `audit_functions.sh` file with stage, rule, merge, reconciliation, lineage, and error functions.

**Add to audit_functions.sh:**

```bash
# ============ SOURCE LIFECYCLE ============

audit_start_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local layer="$1"
  local run_id="$2"
  local batch_id="$3"
  local source_type="$4"   # FILE or TABLE
  local source_name="$5"
  local source_path="${6:-}"
  local file_size="${7:-0}"
  local file_checksum="${8:-}"
  local file_modified_time="${9:-}"
  local watermark_start="${10:-}"
  local watermark_end="${11:-}"
  local input_count="${12:-0}"
  
  local ts=$(_audit_ts)
  # Build INSERT statement for audit_source_control with status=STARTED
  # ... implement similar to audit_start_run
}

# NOTE: source_name is required on every close/fail/partial call. Under Gold fan-in a
# single batch_id has N open source rows (one per Curated input), so batch_id alone no
# longer identifies which row to close. See docs/GOLD_FANOUT_DESIGN.md.

audit_complete_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local run_id="$1"
  local batch_id="$2"
  local source_name="$3"
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
  local source_name="$3"
  local error_message="$4"
  
  # INSERT new row with status=FAILED
}

audit_partial_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local run_id="$1"
  local batch_id="$2"
  local source_name="$3"
  local reason="$4"        # e.g. "Missing Curated inputs: curated_plan,curated_addr"
  
  # INSERT new row with status=PARTIAL and error_message=reason.
  # PARTIAL means "nothing broke, but this unit was not fully processed" — a Gold target
  # skipped by the fan-in gate. It must block a COMPLETED run without being reported as
  # a failure.
}

# ============ STAGE SUMMARY ============

audit_write_stage_summary() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local layer="$1"
  local run_id="$2"
  local batch_id="$3"
  local source_name="$4"
  local stage_name="$5"
  local target_table="$6"
  local input_count="$7"
  local expected_output="$8"
  local actual_written="$9"
  local count_diff="${10}"
  local reject_count="${11:-0}"
  local validation_status="${12}"  # MATCHED or MISMATCHED
  local status="${13}"             # COMPLETED or FAILED
  local start_time="${14}"
  local end_time="${15}"
  local error_message="${16:-}"
  
  local safe_error=$(_escape_sql "$error_message")
  safe_error="${safe_error:0:${AUDIT_ERROR_MSG_MAX_LENGTH}}"
  
  local sql="INSERT INTO ${AUDIT_DB}.audit_stage_summary VALUES (
    '${layer}', '${run_id}', '${batch_id}', '${source_name}', '${stage_name}', '${target_table}',
    ${input_count}, ${expected_output}, ${actual_written}, ${count_diff}, ${reject_count},
    '${validation_status}', '${status}',
    CAST('${start_time}' AS TIMESTAMP), CAST('${end_time}' AS TIMESTAMP),
    CAST((unix_timestamp(CAST('${end_time}' AS TIMESTAMP)) - unix_timestamp(CAST('${start_time}' AS TIMESTAMP))) * 1000 AS BIGINT),
    '${safe_error}', current_timestamp()
  )"
  
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null || {
    fnLogMsg ERROR "audit_write_stage_summary failed for batch_id=${batch_id}"
  }
}

# ============ MERGE SUMMARY ============

audit_write_merge_summary() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  # Parameters: layer, run_id, batch_id, target_table, operation_type,
  #             inserted, updated, deleted, noop, expired,
  #             late_arriving, out_of_order, duplicate,
  #             status, start_time, end_time, error_message
  
  # Build INSERT statement for audit_merge_summary
}

# ============ RECONCILIATION ============

audit_write_reconciliation() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local layer="$1"
  local run_id="$2"
  local entity_name="$3"
  local from_layer="$4"
  local to_layer="$5"
  local source_name="$6"    # source TABLE for this edge, e.g. curated_member
  local target_table="$7"   # target TABLE for this edge, e.g. gold_member_dim
  local source_count="$8"
  local target_count="$9"
  local rejected_count="${10}"
  local filtered_count="${11}"
  local difference="${12}"
  local expected_reason="${13}"
  local unexplained="${14}"
  local status="${15}"  # MATCHED, EXPLAINED, MISMATCHED
  
  local sql="INSERT INTO ${AUDIT_DB}.audit_reconciliation VALUES (...)"
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null
}

# One row per (source table → target table) EDGE. Curated→Gold is many-to-many, so
# (entity_name, from_layer, to_layer) does not identify a reconciliation row: a Curated
# table feeding three Gold tables writes three rows that differ only by target_table.
# source_count / target_count are DISTINCT NATURAL KEYS, not physical rows.

# ============ LINEAGE ============

audit_write_lineage() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local layer="$1"
  local run_id="$2"
  local batch_id="$3"
  local target_table="$4"
  local source_layer="$5"
  local source_run_id="$6"
  local source_batch_id="$7"
  local source_name="$8"
  
  local sql="INSERT INTO ${AUDIT_DB}.audit_lineage VALUES (
    '${layer}', '${run_id}', '${batch_id}', '${target_table}',
    '${source_layer}', '${source_run_id}', '${source_batch_id}', '${source_name}',
    current_timestamp()
  )"
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null
}

# ============ ERROR DETAIL ============

audit_write_error() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local layer="$1"
  local run_id="$2"
  local batch_id="$3"
  local source_name="$4"
  local stage_name="$5"
  local target_table="$6"
  local message_id="$7"
  local record_index="$8"
  local error_type="$9"
  local error_code="${10}"
  local error_message="${11}"
  local record_payload="${12:-}"
  
  local safe_error=$(_escape_sql "$error_message")
  safe_error="${safe_error:0:${AUDIT_ERROR_MSG_MAX_LENGTH}}"
  
  local safe_payload=""
  if [ "${AUDIT_PAYLOAD_ENABLED}" = "true" ] && [ -n "$record_payload" ]; then
    safe_payload=$(_escape_sql "$record_payload")
    safe_payload="${safe_payload:0:${AUDIT_PAYLOAD_MAX_LENGTH}}"
  fi
  
  local sql="INSERT INTO ${AUDIT_DB}.audit_error_detail VALUES (...)"
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null || {
    fnLogMsg ERROR "audit_write_error failed"
  }
}

# ============ GOLD SOURCE MAP HELPERS ============
# Read audit_gold_source_map (prompt 02). Gold instrumentation (prompt 15) and
# reconciliation (prompt 18) drive their loops from these — do not hardcode the mapping.
# Each returns whitespace-separated values on stdout, empty on no match.

audit_get_value() {
  # Like audit_get_count but for non-numeric scalars (table names, roles).
  local query="$1"
  ${AUDIT_BEELINE_CMD} --silent=true --outputformat=tsv2 -e "$query" 2>/dev/null | tail -1
}

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
  # Only this one gets a reconciliation identity; the rest are audited as RI rules.
  local gold_table="$1"
  audit_get_value "SELECT curated_table FROM ${AUDIT_DB}.audit_gold_source_map
                   WHERE gold_table='${gold_table}' AND is_active='Y'
                     AND source_role='DRIVER' LIMIT 1"
}

audit_gold_targets_in_dependency_order() {
  # Gold tables ordered so that anything named in depends_on is built first.
  # Our depends_on graph is one level deep (a fact table depending on its dimensions).
  # If prompt 14 finds deeper nesting, replace this with a real topological sort and say so.
  audit_get_value "SELECT CONCAT_WS(' ', COLLECT_LIST(gold_table)) FROM (
                     SELECT DISTINCT gold_table,
                            CASE WHEN depends_on IS NULL OR depends_on='' THEN 0 ELSE 1 END AS lvl
                     FROM ${AUDIT_DB}.audit_gold_source_map WHERE is_active='Y'
                     ORDER BY lvl, gold_table) t"
}
```

Complete all function implementations with proper INSERT statements.

**Cache the map lookups.** These run per Gold table inside the main loop; a beeline round
trip each is wasteful. Read `audit_gold_source_map` once in `audit_init` into shell
variables or a temp file, and have these helpers read that. If you do, say so in the
output — a stale cache within a run is fine (the map is reference data, not events), but
it must be refreshed per run.

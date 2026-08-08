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

audit_complete_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local run_id="$1"
  local batch_id="$2"
  local input_count="$3"
  local processed_count="$4"
  local rejected_count="$5"
  local completed_targets="$6"
  local failed_targets="$7"
  
  # INSERT new row with status=COMPLETED
}

audit_fail_source() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local run_id="$1"
  local batch_id="$2"
  local error_message="$3"
  
  # INSERT new row with status=FAILED
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
  local source_count="$6"
  local target_count="$7"
  local rejected_count="$8"
  local filtered_count="$9"
  local difference="${10}"
  local expected_reason="${11}"
  local unexplained="${12}"
  local status="${13}"  # MATCHED, EXPLAINED, MISMATCHED
  
  local sql="INSERT INTO ${AUDIT_DB}.audit_reconciliation VALUES (...)"
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null
}

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
```

Complete all function implementations with proper INSERT statements.

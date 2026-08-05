# Prompt S02 — Create shell audit functions (run/source lifecycle)

Create the shell audit library with run and source lifecycle functions.

**Create file:** `audit_functions.sh`

**Existing logging pattern in your scripts:**
[PASTE: Your fnLogMsg function or logging pattern here]

```bash
#!/bin/bash
# Audit Functions Library
# Source this file after audit_config.sh

# ============ INITIALIZATION ============

audit_init() {
  # Source config if not already loaded
  if [ -z "${AUDIT_DB}" ]; then
    source "$(dirname "${BASH_SOURCE[0]}")/audit_config.sh"
  fi
  
  # Check kill switch
  if [ "${AUDIT_ENABLED}" != "true" ]; then
    fnLogMsg INFO "Audit disabled by AUDIT_ENABLED=${AUDIT_ENABLED}"
    return 0
  fi
  
  # Validate beeline connectivity (optional quick check)
  # ${AUDIT_BEELINE_CMD} -e "SELECT 1" >/dev/null 2>&1 || {
  #   fnLogMsg WARN "Audit beeline check failed, continuing anyway"
  # }
  
  fnLogMsg INFO "Audit initialized: db=${AUDIT_DB}"
  return 0
}

# ============ ID GENERATION ============

audit_generate_run_id() {
  # Format: YYYYMMDD_HHMMSS_RANDOM
  echo "$(date +%Y%m%d_%H%M%S)_$(cat /proc/sys/kernel/random/uuid | cut -d'-' -f1)"
}

audit_generate_batch_id() {
  local run_id="$1"
  local table_name="$2"
  echo "${run_id}_${table_name}_$(cat /proc/sys/kernel/random/uuid | cut -d'-' -f1)"
}

# ============ HELPER: ESCAPE SINGLE QUOTES ============

_escape_sql() {
  echo "${1//\'/\'\'}"
}

# ============ HELPER: GET COUNT ============

audit_get_count() {
  local query="$1"
  local result
  result=$(${AUDIT_BEELINE_CMD} --silent=true -e "$query" 2>/dev/null | grep -E '^[0-9]+$' | tail -1)
  echo "${result:-0}"
}

# ============ HELPER: CURRENT TIMESTAMP ============

_audit_ts() {
  date -u +"%Y-%m-%d %H:%M:%S"
}

# ============ RUN LIFECYCLE ============

audit_start_run() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local layer="$1"
  local run_id="$2"
  local pipeline_name="$3"
  local process_name="$4"
  local source_system="$5"
  local spark_app_id="${6:-}"  # May be empty for shell scripts
  
  local ts=$(_audit_ts)
  local sql="INSERT INTO ${AUDIT_DB}.audit_run_control 
    SELECT '${layer}', '${run_id}', '$(_escape_sql "$pipeline_name")', 
           '$(_escape_sql "$process_name")', '${source_system}', '${spark_app_id}',
           'STARTED', CAST('${ts}' AS TIMESTAMP), NULL, NULL,
           0, 0, 0, 0, 0, 0, NULL, current_timestamp()"
  
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null || {
    fnLogMsg ERROR "audit_start_run failed for run_id=${run_id}"
    return 0  # Don't fail the pipeline for audit errors
  }
  fnLogMsg INFO "Audit run started: ${run_id}"
}

audit_complete_run() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local run_id="$1"
  local total_sources="$2"
  local completed_sources="$3"
  local failed_sources="$4"
  local total_input="${5:-0}"
  local total_processed="${6:-0}"
  local total_rejected="${7:-0}"
  
  # Get start_time from the STARTED row
  local start_time=$(${AUDIT_BEELINE_CMD} --silent=true -e "
    SELECT start_time FROM ${AUDIT_DB}.audit_run_control 
    WHERE run_id='${run_id}' AND status='STARTED' LIMIT 1" 2>/dev/null | grep -v "^$" | tail -1)
  
  local ts=$(_audit_ts)
  local sql="INSERT INTO ${AUDIT_DB}.audit_run_control
    SELECT layer, run_id, pipeline_name, process_name, source_system, spark_application_id,
           'COMPLETED', start_time, CAST('${ts}' AS TIMESTAMP), 
           CAST((unix_timestamp(CAST('${ts}' AS TIMESTAMP)) - unix_timestamp(start_time)) * 1000 AS BIGINT),
           ${total_sources}, ${completed_sources}, ${failed_sources},
           ${total_input}, ${total_processed}, ${total_rejected},
           NULL, current_timestamp()
    FROM ${AUDIT_DB}.audit_run_control
    WHERE run_id='${run_id}' AND status='STARTED'"
  
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null || {
    fnLogMsg ERROR "audit_complete_run failed for run_id=${run_id}"
  }
  fnLogMsg INFO "Audit run completed: ${run_id}"
}

audit_fail_run() {
  [ "${AUDIT_ENABLED}" != "true" ] && return 0
  
  local run_id="$1"
  local total_sources="${2:-0}"
  local completed_sources="${3:-0}"
  local failed_sources="${4:-0}"
  local error_message="$5"
  
  local ts=$(_audit_ts)
  local safe_error=$(_escape_sql "$error_message")
  safe_error="${safe_error:0:${AUDIT_ERROR_MSG_MAX_LENGTH}}"
  
  local sql="INSERT INTO ${AUDIT_DB}.audit_run_control
    SELECT layer, run_id, pipeline_name, process_name, source_system, spark_application_id,
           'FAILED', start_time, CAST('${ts}' AS TIMESTAMP),
           CAST((unix_timestamp(CAST('${ts}' AS TIMESTAMP)) - unix_timestamp(start_time)) * 1000 AS BIGINT),
           ${total_sources}, ${completed_sources}, ${failed_sources},
           0, 0, 0, '${safe_error}', current_timestamp()
    FROM ${AUDIT_DB}.audit_run_control
    WHERE run_id='${run_id}' AND status='STARTED'"
  
  ${AUDIT_BEELINE_CMD} -e "$sql" 2>/dev/null || {
    fnLogMsg ERROR "audit_fail_run failed for run_id=${run_id}"
  }
  fnLogMsg INFO "Audit run failed: ${run_id}"
}
```

Continue with source lifecycle methods in the same file (audit_start_source, audit_complete_source, audit_fail_source).

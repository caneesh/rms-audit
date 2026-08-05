# Prompt S01 — Create shell audit config

Create a shell config file for the audit framework. This will be sourced by all Curated and Gold scripts.

**Create file:** `audit_config.sh`

```bash
#!/bin/bash
# Audit Framework Configuration
# Source this file in your pipeline scripts

# ============ MASTER KILL SWITCH ============
# Set to "false" to disable all audit writes
AUDIT_ENABLED="${AUDIT_ENABLED:-true}"

# ============ DATABASE AND PATHS ============
AUDIT_DB="${AUDIT_DB:-audit}"
AUDIT_STAGING_ROOT="${AUDIT_STAGING_ROOT:-/data/staging/audit}"

# ============ BEELINE CONNECTION ============
# Adjust to match your existing hivebeeline wrapper or connection
AUDIT_BEELINE_CMD="${AUDIT_BEELINE_CMD:-hivebeeline}"

# ============ RETENTION ============
AUDIT_STAGING_RETENTION_POLICY="${AUDIT_STAGING_RETENTION_POLICY:-delete-on-success}"
AUDIT_STAGING_RETENTION_DAYS="${AUDIT_STAGING_RETENTION_DAYS:-7}"

# ============ RERUN BEHAVIOR ============
AUDIT_FORCE_RERUN="${AUDIT_FORCE_RERUN:-false}"
AUDIT_INFLIGHT_STALENESS_MINUTES="${AUDIT_INFLIGHT_STALENESS_MINUTES:-60}"

# ============ ERROR CAPTURE LIMITS ============
AUDIT_ERROR_MSG_MAX_LENGTH="${AUDIT_ERROR_MSG_MAX_LENGTH:-4000}"
AUDIT_PAYLOAD_ENABLED="${AUDIT_PAYLOAD_ENABLED:-false}"  # PHI risk - default OFF
AUDIT_PAYLOAD_MAX_LENGTH="${AUDIT_PAYLOAD_MAX_LENGTH:-1000}"
AUDIT_SAMPLE_KEY_CAP="${AUDIT_SAMPLE_KEY_CAP:-20}"
AUDIT_ERROR_BUFFER_MAX="${AUDIT_ERROR_BUFFER_MAX:-10000}"

# ============ RULE THRESHOLDS ============
AUDIT_DEFAULT_THRESHOLD_PCT="${AUDIT_DEFAULT_THRESHOLD_PCT:-0}"

# ============ BLOCKING RULES ============
# Comma-separated list of rule names that fail the pipeline
AUDIT_BLOCKING_RULES="${AUDIT_BLOCKING_RULES:-NULL_MID,NULL_CORP}"

# ============ HQL TEMPLATES PATH ============
AUDIT_HQL_PATH="${AUDIT_HQL_PATH:-/app/audit/hql}"
```

Also create a template file `audit_config.sh.template` with placeholders for site-specific values.

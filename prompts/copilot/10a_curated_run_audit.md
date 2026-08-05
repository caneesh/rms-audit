# Prompt 10a — Add Curated run audit

Add audit instrumentation to the Curated master shell script for run-level tracking.

**Context:**
- Shell audit functions exist (from prompts S02/S03)
- audit_config.sh exists (from S01)
- This is the Curated layer (layer = "CURATED")

**Existing patterns from your scripts:**
- Logging: `fnLogMsg INFO/ERROR "message"`
- Beeline: `hivebeeline -f $hqlFile --hivevar ...`
- Error check: `if [ $? -ne 0 ]; then ... fi`

**Add at the START of the master script:**

```bash
#!/bin/bash
# ... existing setup code ...

# Source audit library
source /path/to/audit_config.sh
source /path/to/audit_functions.sh

# Initialize audit
audit_init

# Generate run ID
RUN_ID=$(audit_generate_run_id)
export RUN_ID  # Make available to child scripts

# Start run audit
audit_start_run "CURATED" "${RUN_ID}" "CuratedPipeline" "CDC_Load" "RMS"

# Track totals
TOTAL_SOURCES=0
COMPLETED_SOURCES=0
FAILED_SOURCES=0

# ... existing table loop will go here (prompt 10b) ...
```

**Add at the END of the master script:**

```bash
# ... after table loop ...

# Complete or fail run
if [ ${FAILED_SOURCES} -gt 0 ]; then
  audit_fail_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "${FAILED_SOURCES}" \
    "Failed sources: ${FAILED_SOURCES}"
  fnLogMsg ERROR "Curated run FAILED: ${RUN_ID}"
  exit 1
else
  audit_complete_run "${RUN_ID}" "${TOTAL_SOURCES}" "${COMPLETED_SOURCES}" "0"
  fnLogMsg INFO "Curated run COMPLETED: ${RUN_ID}"
fi
```

Show the diff of changes to your master script.

[PASTE: Your Curated master script here]

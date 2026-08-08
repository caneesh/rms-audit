# Prompt 03b — Shell audit helper library

Create a reusable shell library (`audit_functions.sh`) that all Curated and Gold scripts
will source. This is the shell equivalent of the Scala AuditWriter.

## Core functions

```
audit_init           — Source config, validate beeline connectivity, set AUDIT_ENABLED
audit_generate_run_id    — Generate UUID-style run_id (date + random or /proc/sys/kernel/random/uuid)
audit_generate_batch_id  — Generate batch_id (run_id + table + sequence)

audit_start_run      — INSERT into audit_run_control (status=STARTED)
audit_complete_run   — INSERT into audit_run_control (status=COMPLETED, totals)
audit_fail_run       — INSERT into audit_run_control (status=FAILED, error_message)

audit_start_source   — INSERT into audit_source_control (status=STARTED)
audit_complete_source — INSERT into audit_source_control (status=COMPLETED, counts)
audit_fail_source    — INSERT into audit_source_control (status=FAILED)

audit_write_stage_summary — INSERT into audit_stage_summary
audit_write_rule_results  — INSERT into audit_rule_result (batch of rules)
audit_write_merge_summary — INSERT into audit_merge_summary
audit_write_reconciliation — INSERT into audit_reconciliation
audit_write_lineage       — INSERT into audit_lineage
audit_write_error         — INSERT into audit_error_detail

audit_get_count      — Execute COUNT(*) query, return result to caller
audit_check_prior_run — Query audit_source_control for prior status (for rerun logic)
```

## Rules

1. **Kill switch**: If `AUDIT_ENABLED=false`, every function is a no-op (return 0 immediately).
   Check this ONCE in `audit_init`, set a flag, and test it at function entry.

2. **Beeline execution**: Use the existing `hivebeeline` wrapper. Construct INSERT statements
   with proper quoting. Use HEREDOCs for multi-line SQL:
   ```bash
   hivebeeline -e "$(cat <<EOF
   INSERT INTO ${AUDIT_DB}.audit_run_control
   SELECT '${layer}', '${run_id}', ...
   EOF
   )"
   ```

3. **Error handling**: Audit failures must NEVER hide business failures.
   - If a business operation failed and audit_fail_run also fails, log the audit failure
     but exit with the ORIGINAL business error code.
   - Audit functions return 0 on success, 1 on failure, but callers should NOT exit on
     audit failure alone (log and continue).

4. **Logging integration**: Use the existing `fnLogMsg INFO/ERROR` pattern. Every audit
   function logs what it's doing at INFO level, errors at ERROR level.

5. **Timestamps**: Use `date -u +"%Y-%m-%d %H:%M:%S"` for UTC timestamps. Never rely on
   session timezone.

6. **String escaping**: Escape single quotes in error messages before INSERT
   (`${msg//\'/\'\'}` or a helper function).

7. **Config sourcing**: `audit_init` sources an audit config file that defines:
   - AUDIT_ENABLED (true/false)
   - AUDIT_DB (database name)
   - AUDIT_ERROR_MSG_MAX_LENGTH (default 4000)
   - AUDIT_PAYLOAD_ENABLED (default false)
   - AUDIT_PAYLOAD_MAX_LENGTH (default 1000)
   - AUDIT_SAMPLE_KEY_CAP (default 20)

## Counting helper

`audit_get_count` executes a COUNT query and captures the result:
```bash
audit_get_count() {
  local query="$1"
  local result
  result=$(hivebeeline -e "$query" 2>/dev/null | grep -E '^[0-9]+$' | tail -1)
  echo "${result:-0}"
}
```
Callers use: `input_count=$(audit_get_count "SELECT COUNT(*) FROM ${table} WHERE ...")`

## Deliverables

1. `audit_functions.sh` — the complete library
2. `audit_config.sh.template` — example config file
3. Usage example showing: init → start_run → start_source → stage → complete
4. Documentation of every function's parameters

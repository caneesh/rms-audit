# Prompt 08 — Raw rerun and duplicate protection

Implement rerun protection based on audit_source_control (append-only: always evaluate
the LATEST row per file identity).

File identity = source path + file name + size + modification time (+ checksum if
prompt 05 captured it).

Before processing a file:
- Latest status COMPLETED → skip the file (log + write a stage-less audit note), unless
  force-rerun is enabled.
- Latest status FAILED or PARTIAL → process as a retry: new batch_id,
  attempt_number = prior + 1, prior_batch_id = the failed batch. Never overwrite or
  delete prior audit rows.
- Latest status STARTED/VALIDATING (in-flight or crashed) → configurable policy:
  treat as crashed after a configurable staleness window, else refuse to double-process.
  State the assumption about concurrent runs found in prompt 01.

Retry of a PARTIAL file:
- Read audit_stage_summary for the prior batch. Skip targets whose latest stage row is
  COMPLETED (their data is already committed). Re-run only FAILED / never-run targets.
- New staging paths come free (path includes the new batch_id).
- Document the one unrecoverable case honestly: a crash DURING a final commit may leave
  partial rows in that one Raw table that cannot be identified or removed (frozen schema).
  Detection: stage row stuck in STARTED with error_type='COMMIT_FAILURE' or no terminal
  row. Write the manual-cleanup runbook note for this case; do not pretend to automate it.

Force-rerun of a COMPLETED file will append duplicate rows to Raw (schema is frozen; we
cannot delete the originals). Log a loud warning stating exactly this when force-rerun is
used, and record the rerun in audit as a new attempt.

State transitions to implement:
STARTED → VALIDATING → COMPLETED | FAILED | PARTIAL

# Prompt 03 — Scala audit models + audit configuration

Using the Scala/Spark versions found in prompt 01, create:

## Case classes (match the prompt-02 DDL exactly — names and types)
- AuditRunControl, AuditSourceControl, AuditStageSummary, AuditRuleResult,
  AuditMergeSummary, AuditReconciliation, AuditLineage, AuditErrorDetail
- RawTarget (name, dataframe supplier, final table, staging path) — used in prompt 06.

Rules:
- java.sql.Timestamp for TIMESTAMP, Long for BIGINT, Int for INT, Double for DOUBLE.
- Option[] only where Spark encoders in our version handle it safely.
- Companion factory methods for STARTED / COMPLETED / FAILED / PARTIAL rows where useful
  (remember: each transition is a NEW row, so factories, not mutators).
- Do not touch existing business models.

## AuditConfig (single config class, wired into the existing config mechanism)
- audit.enabled (master kill switch — false must restore today's exact behavior)
- audit database name
- staging root path
- staging retention policy (delete-on-success | keep-N-days)
- force-rerun flag
- error_message max length, payload capture on/off (default OFF), payload max length,
  masking rule reference
- sample_failed_keys cap (default 20)

Show the proposed package structure and file list before writing the implementation.

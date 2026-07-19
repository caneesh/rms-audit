# Prompt 16 — Gold rules, referential integrity, and history audit

## Rule results (audit_rule_result, layer='GOLD')
- BUSINESS rules from prompt 14 (current coverage, latest member, primary address,
  subscriber selection, active product, effective-date logic).
- DQ rules: missing MID, duplicate member, expired coverage on active record, invalid
  product, missing subscriber, bad effective date (from prompt 14 inventory).
- RI rules (rule_type='RI'): member exists, coverage exists, provider exists, product
  exists. Implement as left-anti joins against the referenced entity, counted in the
  same aggregation pass where possible; missing-reference sample keys capped per config.
- Mandatory one-pass counting discipline, thresholds, blocking-vs-warning config, and
  batched writeRuleResults — same as prompt 11.

## History / SCD / PIT audit (audit_merge_summary, layer='GOLD')
Per Gold target, from the DataFrames the SCD logic already computes (no extra target
scans):
- SCD1: updated_count. SCD2: inserted_count (new versions), expired_count (closed
  versions), noop_count (ignored/unchanged).
- History corrections (effective-date changes to existing versions): updated_count with
  operation_type='SCD2', plus an error_detail row of type 'HISTORY_CORRECTION' if the
  correction is unexpected per business rules.
- PIT tables if present: operation_type='PIT' with created/updated/closed counts and the
  snapshot timestamp recorded in the row (use start_time for the snapshot ts and say so
  in a comment).

A blocking rule failure or history-apply failure fails that target's stage.

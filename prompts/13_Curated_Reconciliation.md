# Prompt 13 — Raw → Curated reconciliation

Implement audit_reconciliation for each Curated entity.

The invariant is an accounting identity, not raw_count == curated_count:

  source_count = target_count + rejected_count + filtered_count(with named reasons)

- source_count: Raw rows in scope for this run (already counted in source_control — reuse,
  do not recount).
- target_count: Curated rows written this batch (from stage_summary / _audit_batch_id).
- rejected_count: from rule failures that reject rows.
- filtered_count: sum of the intentional drops inventoried in prompt 09, each with its
  named reason concatenated into expected_difference_reason (e.g.
  'INACTIVE_MEMBER:1200;DEDUP:45').
- unexplained_difference = source - target - rejected - filtered.

Status: MATCHED (diff 0, nothing filtered), EXPLAINED (diff fully accounted for),
MISMATCHED (unexplained_difference != 0).

Policy: MISMATCHED fails the run (this is the Curated completion gate for counts).
All inputs to this identity must come from numbers already computed during the run —
this step performs NO new scans of Raw or Curated tables.

One reconciliation row per entity per run, from_layer='RAW', to_layer='CURATED'.

Finally: implement the Curated completion gate — a last step that reads this run's audit
rows (sources completed, blocking rules passed, merges succeeded, reconciliation not
MISMATCHED, no fatal errors) and writes the final run_control status accordingly.

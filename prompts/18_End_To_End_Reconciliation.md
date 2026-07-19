# Prompt 18 — Curated → Gold reconciliation and end-to-end view

## Curated → Gold reconciliation (audit_reconciliation)
Same accounting-identity approach as prompt 13, per entity, from_layer='CURATED',
to_layer='GOLD', with two Gold-specific twists:
- Fan-out is expected: one Curated row may legitimately produce rows in multiple Gold
  tables, and SCD2 writes version rows. Define the per-entity expected relationship
  explicitly (e.g., 'curated members in scope = gold member natural keys touched this
  batch', NOT raw row counts) and record it in expected_difference_reason.
- All counts come from numbers already computed during the run (stage_summary,
  merge_summary, rule_result). No new table scans.
MISMATCHED (unexplained difference) fails the Gold run.

## Gold completion gate
Final step reads this run's audit rows and marks the run COMPLETED only when:
all Curated inputs processed, blocking rules passed, history/merge succeeded,
RI checks passed (or within threshold), reconciliation not MISMATCHED, lineage written,
no fatal errors. Otherwise FAILED/PARTIAL with reasons in error_message.

## End-to-end reconciliation view
Create a SQL view (or query in the ops pack) that joins the RAW→CURATED and
CURATED→GOLD reconciliation rows per entity per day into one line:
raw_count, curated_count, gold_count, explained diffs, unexplained diffs, overall status.
This is the single row support looks at each morning per entity.

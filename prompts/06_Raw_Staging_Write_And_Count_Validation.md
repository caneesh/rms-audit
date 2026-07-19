# Prompt 06 — Raw staging write + count validation for all 13 tables

Replace the 13 direct Raw writes with ONE reusable write-with-audit function driven by a
RawTarget collection. Do not duplicate logic 13 times. Do not change DataFrame schemas,
transformations, final table names, final storage format, or final partitioning.

## Design decision (already made — do not revisit)
Raw table schemas are frozen, so rows can never be identified in the final table after an
append. Therefore ALL validation happens in staging:
`actual_written_count` = rows counted in the staging output. The final commit is trusted,
not re-verified. Do not implement any before/after count of the final Raw table.

## Per target, the function must:
1. Record stage start; persist the DataFrame (appropriate storage level).
2. expected_output_count = df.count() on the persisted DataFrame (count once).
3. Write to the staging path: stagingRoot/run_id/batch_id/target_table/, using the SAME
   format the final table uses.
4. Read the staged output back and count → actual_written_count.
5. If mismatch: write error_detail (error_type='COUNT_MISMATCH'), write stage_summary
   with MISMATCHED/FAILED, do NOT commit, and fail this stage.
6. If match: commit staged data to the final Raw destination with the existing production
   write behavior (mode, format, partitioning preserved exactly).
7. If the commit itself throws: stage FAILED with error_type='COMMIT_FAILURE'; staging is
   retained for troubleshooting per config.
8. Write stage_summary; unpersist in a finally block.
9. Clean staging after successful commit, or retain per AuditConfig retention policy.

## File-level outcome
- All 13 stages COMPLETED → completeSource (status COMPLETED).
- Some completed, some failed → markSourcePartial, with completed/failed target counts.
- A stage failure stops remaining stages for that file (fail fast) but not other files.

List all 13 target mappings (DataFrame → staging path → final table) in the final response.

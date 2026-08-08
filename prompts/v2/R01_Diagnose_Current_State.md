# R01 — Diagnose the current state (READ ONLY)

**Make no edits. Change nothing in Hive. This prompt only looks and reports.**

Later v2 prompts patch code that was generated from `prompts/02`, `03b`, `10`, `11` and
`13`. They assume that code has the shape those prompts described — but Copilot may have
deviated, and the DDL that was actually run against Hive may not match the DDL in the repo.
Establish the truth before changing anything.

## Baseline

Everything under Phase 0 (foundation), Phase 1 (Raw/Scala audit) and Phase 2 (Curated) is
believed complete. Gold has not been started.

## What to find

Write the answers to **`docs/AUDIT_V2_BASELINE.md`**. For anything you cannot find, write
**NOT FOUND** — do not guess, and do not infer a path from the prompts.

### 1. Files on disk

Locate and record the full path of each:

- the shell audit library (`audit_functions.sh` or whatever it was named)
- the shell audit config (`audit_config.sh`)
- the audit DDL `.sql` files, and the master DDL file
- the audit rule HQL templates
- the Curated master script and the table-loop script(s) that were instrumented
- the Curated reconciliation logic — note whether it was implemented in shell, in HQL, or both
- the Scala audit models and `AuditWriter`

### 2. Actual shell function signatures

For each of these, copy the **real parameter list** as generated, in order:

- `audit_start_source`
- `audit_complete_source`
- `audit_fail_source`
- `audit_write_reconciliation`
- `audit_write_stage_summary`
- `audit_write_lineage`
- `audit_get_count`

Also list any function the library defines that is **not** in `prompts/03b`, and any
function `prompts/03b` lists that is **missing**.

This matters because R04 changes three of these signatures by inserting parameters. If the
generated signatures already differ from the prompts, R04's instructions need adjusting
before they are applied.

### 3. Scala side

- Does an `AuditReconciliation` model exist? Copy its field list in order.
- Does `AuditWriter` have a `writeReconciliation` method? Is it called anywhere, or only
  defined? (Raw's reconciliation policy writes an error row rather than a reconciliation
  row, so it may be unused.)
- Is there a test that compares the models against the DDL files — the drift test from
  `prompts/20`? If so, give its path.

### 4. What is actually deployed in Hive

For the audit database, run and capture:

```sql
SHOW TABLES IN ${AUDIT_DB};
DESCRIBE FORMATTED ${AUDIT_DB}.audit_reconciliation;
DESCRIBE FORMATTED ${AUDIT_DB}.audit_source_control;
```

Compare the deployed `audit_reconciliation` column list against the DDL file in the repo and
report any difference. Confirm whether `audit_gold_source_map` exists (it should not yet).

### 5. Has Curated ever run?

```sql
SELECT layer, status, COUNT(*)
FROM ${AUDIT_DB}.audit_run_control
GROUP BY layer, status;

SELECT layer, COUNT(*) AS row_count, SUM(CASE WHEN status='MISMATCHED' THEN 1 ELSE 0 END) AS mismatched
FROM ${AUDIT_DB}.audit_reconciliation
GROUP BY layer;

-- The signature of the double-count bug: a run that otherwise succeeded, reporting a
-- large unexplained difference — often close to the source count itself.
SELECT run_id, entity_name, source_count, target_count,
       rejected_count, filtered_count, unexplained_difference, status
FROM ${AUDIT_DB}.audit_reconciliation
WHERE layer = 'CURATED'
ORDER BY created_ts DESC
LIMIT 20;
```

Report exactly what came back. If there are rows with a large `unexplained_difference` on
runs that were otherwise healthy, that confirms the bug R05 fixes, and R06 can use these
numbers as its before/after comparison. If `audit_reconciliation` is empty, say so — the bug
is latent and R05 fixes it before first exposure.

Also check whether MISMATCHED is actually wired to fail the run, or only logged. A pipeline
that computes a mismatch and continues anyway would explain healthy-looking runs.

### 6. Anything that will make the R02–R05 block harder

Call out anything you notice that the v2 prompts do not anticipate. Specifically:

- Curated instrumentation copy-pasted into several scripts rather than shared, so R05's
  call-site changes have more places to reach.
- Any Hive `UPDATE` statement against an audit table — that breaks the append-only rule and
  needs raising separately.
- Any audit code outside the paths listed in section 1.

## Output

`docs/AUDIT_V2_BASELINE.md`, structured under the six headings above.

End it with a short **Readiness** section: is it safe to proceed to the R02–R05 block, and
if not, what needs deciding first.

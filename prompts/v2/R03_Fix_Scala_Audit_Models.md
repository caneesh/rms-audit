# R03 — Fix the Scala audit models

**Part of the R02–R05 atomic block. Do not run any pipeline until R05 is complete.**

## Baseline

Patches the output of `prompts/03_Create_Audit_Models_And_Config.md` and
`prompts/04_Create_AuditWriter.md`. Use `docs/AUDIT_V2_BASELINE.md` (R01) section 3 for the
real paths and the current field list.

The Raw layer's audit instrumentation is **already built and deployed**, and it shares the
audit tables with Curated and Gold. R02 just changed `audit_reconciliation`, so the Scala
side is now out of sync.

## Why this is needed even if Raw never writes reconciliation rows

Raw's reconciliation policy (`prompts/05`) writes an **error row** when
`input_count != processed + rejected` — it may never call `writeReconciliation` at all. That
does not make this optional:

- `prompts/20` specifies a test that reads the DDL files and compares them against the case
  classes so that drift fails the build. If that test exists, it is failing right now.
- An unused model that silently disagrees with the table is a trap for whoever writes the
  next caller.

So update the model regardless of whether anything calls it. If R01 found no drift test,
note that in the output — it is Phase 4 debt, not something to build here.

## Changes

### 1. `AuditReconciliation` model

Add two fields immediately after `toLayer`, matching the DDL column order exactly:

```scala
sourceName: String,      // source table for this edge
targetTable: String,     // target table for this edge
```

The **order matters**, not just the presence — the writer inserts positionally.

### 2. `AuditWriter.writeReconciliation`

Update the signature and the INSERT column list to match. Keep the method's existing
error-handling contract unchanged: if the audit write throws, log it and re-throw the
original business exception (rule 2).

### 3. Any call sites

Search the Scala source for `writeReconciliation` and `AuditReconciliation`. Update each.
There may be none — say so explicitly rather than leaving it ambiguous.

### 4. The drift test

If R01 found a model-vs-DDL test, run it and confirm it passes against the R02 schema. If it
compares column names, it will need the two new names; if it compares counts, it will need
the new count.

## What not to change

- Do **not** touch the Raw pipeline's business logic, DataFrame schemas, or output.
- Do **not** add `layer`-style columns to the model for `audit_gold_source_map` — that table
  is shell-side reference data and has no Scala model.
- Do **not** change any other audit model. Only `audit_reconciliation` changed in R02.

## Deliverables

1. Updated `AuditReconciliation` case class, shown as a diff.
2. Updated `AuditWriter.writeReconciliation`, shown as a diff.
3. A list of call sites updated, or an explicit statement that there were none.
4. Drift-test result: passing, or "no drift test exists" recorded as Phase 4 debt.
5. Confirmation that the project still compiles.

## Do not stop here

R04 must follow — the shell library still has the old signatures, and until R05 is done the
Curated callers do not match either.

# Prompt 17 — Gold lineage (batch-level + traceability keys)

Implement lineage so any Gold record can be traced to its MQ-origin file. Two parts:

## 1. Batch-level edges (audit_lineage)
For every Gold target written, one row per contributing Curated source batch:
layer='GOLD', run/batch of the Gold write, source_layer='CURATED', source_run_id/
source_batch_id/source_name of the Curated input. (The Curated side already wrote its
CURATED→RAW edges in prompt 10, and Raw's source_control links batches to files.)
Do NOT create any row-level lineage table. Cardinality of audit_lineage is
batches × sources — tiny.

## 2. Row-level traceability by joining (document, don't materialize)
Write a short doc section (docs/LINEAGE.md) showing the actual join chain, using the
real keys found in prompt 14:

  gold_row._audit_batch_id → audit_lineage → curated batch
  → curated rows (_audit_batch_id + business key) → raw rows (business/transaction key)
  → audit_source_control (raw batch) → source_file_name → MQ origin

Include one worked example query: given a Gold member row, return the source file name
and run that produced it. Note honestly where a hop is business-key-based (Raw has no
audit columns — its hop uses the transaction-header/natural key found in prompt 01/14).

## Completion gate input
Lineage rows written successfully is one of the Gold completion criteria (prompt 18).

# Prompt 14 — Analyze the Gold pipeline

Analyze the Gold pipeline implementation. **Do not make any code changes.**

Identify and report:
1. Entry points and flow per Gold table. The Curated → Gold mapping (which Curated
   tables feed which Gold tables — expected to be one-to-many).
2. How history is maintained: SCD type per table (1/2/other), PIT tables if any,
   effective/expiry date handling, how versions are closed.
3. The key business rules: current coverage, latest member, primary address, subscriber
   selection, active product, effective-date logic — name what actually exists in code.
4. Referential-integrity relationships worth checking (member exists, coverage exists,
   provider exists, product exists — confirm against actual keys).
5. Every intentional drop/filter/dedup (named reconciliation reasons, as in prompt 09).
6. Whether Gold schemas can accept _audit_run_id / _audit_batch_id columns.
7. How Gold writes physically happen (partition overwrite? append? rewrite?).
8. Natural keys that connect a Gold row back to Curated and Raw rows (member id,
   transaction header id, etc.) — needed for row-level traceability by joining.

Output: mappings, SCD/PIT inventory, rule inventory, RI list, drop inventory,
schema-change feasibility, write mechanism, traceability keys, risks. No code.

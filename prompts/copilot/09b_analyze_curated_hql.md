# Prompt 09b — Analyze Curated HQL files

Continuing Curated analysis. Now look at the HiveQL files.

**Do not write any code yet. Analysis only.**

Tell me:
1. For each Curated table: which Raw tables it reads from
2. How incremental scope is determined (timestamp column? partition? full reload?)
3. The CDC/merge mechanism: MERGE INTO? INSERT OVERWRITE? DELETE + INSERT?
4. Where records are dropped/filtered and why (list each with business reason)
5. What business rules or DQ checks exist in the HQL
6. The important JOINs (risk of row explosion or silent drops)
7. Can Curated schemas accept two new columns: `_audit_run_id`, `_audit_batch_id`?

[PASTE: One or two Curated HQL files (the merge or CDC logic)]

[PASTE: Any transformation/filtering HQL]

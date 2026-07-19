# Prompt 09 — Analyze the Curated pipeline

Analyze the Curated pipeline implementation. **Do not make any code changes.**

Identify and report:
1. Entry points and one flow description per Curated table.
2. For each Curated table: which Raw tables it reads (the source mapping), and how
   incremental scope is determined (watermark, CDC timestamp, partition, full reload?).
3. How change data is applied to Curated tables — exactly. Hive ACID? INSERT OVERWRITE
   of partitions? Full rewrite? Something else? (Our Spark has no MERGE INTO — find what
   the code really does; the merge-audit design in prompt 12 depends on this.)
4. Every place records are dropped, deduplicated, filtered, or rejected — list each with
   the business reason if discernible. These become named reconciliation reasons.
5. The important business rules and DQ checks already in the code (name them).
6. The two or three load-bearing joins (highest risk of row explosion or silent drops).
7. Whether Curated schemas are owned by this team and can accept two new columns
   (_audit_run_id, _audit_batch_id).
8. Existing config, logging, and error handling patterns.

Output: per-table source mappings, CDC/apply mechanism, the drop/filter inventory, rule
inventory, join list, schema-change feasibility, risks. No code.

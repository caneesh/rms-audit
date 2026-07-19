# Prompt 01 — Analyze the existing Raw load (XMLToHive)

Analyze the complete XMLToHive Raw Load implementation. **Do not make any code changes.**

Identify and report:
1. Main entry point and overall processing flow.
2. Exact Scala, Spark, and Hive versions from the build files. All later work must
   target these versions — report them explicitly.
3. Sequence File reading logic (how files are listed, opened, and iterated).
4. The XML parsing / processJson flow and where messages are rejected today.
5. How each of the 13 Raw DataFrames is created, and for each: target table name,
   write mode, storage format, partitioning, HDFS location.
6. Whether Raw entity rows carry a transaction-header reference or any natural key that
   links a row back to its source message (needed later for traceability).
7. Existing exception handling and logging patterns.
8. Existing configuration mechanism (how table names / paths / settings are supplied).
9. Whether two runs can ever execute concurrently (scheduler behavior if visible).
10. Repository conventions for Hive DDL scripts and where new DDL should live.

Recommend (design only, no code):
- Insertion points for run-level, file-level, stage-level, and error-level audit calls.
- A staging path convention for validated writes: staging root / run_id / batch_id / table.
- Risks or compatibility concerns for adding auditing.

Output: flow description, impacted file list, the 13 table mappings, version report,
recommended insertion points, risks. Nothing else.

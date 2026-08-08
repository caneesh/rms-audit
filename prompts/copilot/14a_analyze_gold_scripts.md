# Prompt 14a — Analyze Gold shell scripts

I'm adding audit instrumentation to the Gold layer shell scripts.

**Do not write any code yet. Analysis only.**

Look at the Gold scripts and tell me:
1. Which script is the entry point / master script?
2. The purpose of each script in the directory:
   - `gold_mbrshp_rms_cdckeys_load.sh`
   - `gold_mbrshp_rms_common_hiveMerge_withGoldDB.sh`
   - `gold_mbrshp_rms_history_table.sh`
   - `gold_mbrshp_rms_create_cdc_trigger.sh`
   - etc.
3. The call graph: which scripts call which?
4. How Gold knows Curated is complete (trigger files?)
5. How SCD2/history is implemented (version closing, effective dates)
6. **Curated → Gold table mapping.** This is the most important answer here — prompts 15a
   and 18b both key off it. The relationship is many-to-many: one Gold table is built from
   several Curated tables (fan-in) *and* one Curated table feeds several Gold tables
   (fan-out). Give me one row per edge:

   | gold_table | curated_table | source_role | join_key | expected_cardinality | depends_on |
   |---|---|---|---|---|---|

   `source_role` is **DRIVER** if that table's rows determine the Gold table's row
   population (usually the one in the `FROM` clause of the main SELECT), **ENRICH** if it
   adds columns to driver rows, **LOOKUP** if it is a code/dimension join. Flag any Gold
   table with two or more genuine drivers — a UNION or a FULL OUTER JOIN — separately.

7. The real values for these, with the file and line you found them in. Say **NOT FOUND**
   rather than guessing: the Gold table list variable, the Hive database names for Curated
   and Gold, the watermark/window predicate, the Gold partition column, the `hivebeeline`
   wrapper's exact name and flags, the `fnLogMsg` signature, and the column the SCD logic
   sets to mark insert/update/expire.

8. The **natural key** per Gold entity — reconciliation counts distinct keys, not rows.

## Save your answer to a file

Write all of the above to `docs/GOLD_ANALYSIS.md`, and the item-6 table as INSERT
statements in `sql/audit_gold_source_map_seed.sql`. Prompt 15a and everything after it
reads those files — they will not have this conversation.

[PASTE: Your Gold master script]

[PASTE: The hiveMerge or SCD script]

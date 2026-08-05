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
6. Curated → Gold table mapping (which Curated tables feed which Gold tables)

[PASTE: Your Gold master script]

[PASTE: The hiveMerge or SCD script]

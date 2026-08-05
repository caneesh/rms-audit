# Prompt 09a — Analyze Curated shell scripts

I'm adding audit instrumentation to shell scripts that run the Curated layer. These scripts call `hivebeeline` to execute HiveQL.

**Do not write any code yet. Analysis only.**

Look at the master script and tell me:
1. The entry point script name
2. How it iterates over tables (loop structure, where table list comes from)
3. How `.prm` parameter files are sourced
4. How `hivebeeline` is called (wrapper function? direct call?)
5. What `--hivevar` parameters are passed
6. How trigger files from Raw are detected
7. How errors are handled (exit on first error? continue?)
8. The logging pattern (`fnLogMsg` or similar)

[PASTE: Your Curated master script here]

[PASTE: One example .prm parameter file]

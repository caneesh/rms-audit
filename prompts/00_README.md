# RMS AUDIT — PROMPT PACK (Raw + Curated + Gold, unified model)

Run the prompts in numerical order, one at a time. Review each result before the next.
Read `docs/AUDIT_DEVELOPER_GUIDE.md` first — the prompts assume its rules.

## Architecture note

- **Raw layer**: Scala/Spark program (XMLToHive) — uses AuditWriter (Scala)
- **Curated/Gold layers**: Shell scripts calling `hivebeeline` — uses audit_functions.sh

Prompts 03/04 are for Scala. Prompts 03b/04b are for shell scripts.
Both write to the same audit tables (prompt 02 DDL).

## Scope and non-negotiables (repeat these to the assistant if it drifts)

- One shared audit schema for all three layers (`layer` column), 8 tables.
- Raw table schemas are FROZEN. Curated/Gold get only `_audit_run_id`, `_audit_batch_id`.
- Audit tables are APPEND-ONLY. Status = latest row per key. No Hive UPDATE, ever.
- Audit failures must never hide business failures.
- `audit.enabled=false` (Scala) or `AUDIT_ENABLED=false` (shell) restores today's exact
  behavior (kill switch).
- No PHI in audit tables by default; payload capture off unless configured, always truncated.
- Match the project's actual Scala / Spark / Hive versions (prompt 01 discovers them —
  do not assume).

## Phases

| Phase | Prompts | Description |
|-------|---------|-------------|
| Phase 0 | 01–04 | Foundation (analysis + shared components) |
| Phase 0b | 03b–04b | Shell audit library (for Curated/Gold) |
| Phase 1 | 05–08 | Raw layer (Scala) |
| Phase 2 | 09–13 | Curated layer (shell/beeline) |
| Phase 3 | 14–18 | Gold layer (shell/beeline) |
| Phase 4 | 19–21 | Wrap-up (ops queries, tests, final review) |

## Execution order

```
01  Analyze Raw Pipeline (Scala)
02  Create Unified Audit Schema DDL
03  Create Audit Models and Config (Scala)
04  Create AuditWriter (Scala)
03b Create Shell Audit Library
04b Create HiveQL Audit Templates

05  Raw Run and File Audit
06  Raw Staging Write and Count Validation
07  Raw Error Auditing
08  Raw Rerun and Duplicate Protection

09  Analyze Curated Pipeline (shell/beeline)
10  Curated Run and Source Audit
11  Curated Rule Results
12  Curated CDC Merge Audit
13  Curated Reconciliation

14  Analyze Gold Pipeline (shell/beeline)
15  Gold Run Source and Entity Audit
16  Gold Rules RI and History Audit
17  Gold Lineage
18  End to End Reconciliation

19  Operational Query Pack
20  Tests
21  Final Review
```

## Tips

- Prompts 01, 09, 14 are analyze-only. Do not let the assistant write code during them.
- If a later prompt needs a decision (e.g., staging path root, audit DB name), make the
  decision yourself and state it in the prompt — don't let the assistant guess.
- For shell prompts (09-18), share actual script snippets or screenshots to help the
  assistant understand the existing patterns (`fnLogMsg`, `hivebeeline`, `.prm` files).
- The shell audit library (03b) must be created BEFORE Curated/Gold instrumentation.

## Key differences: Scala vs Shell

| Aspect | Scala (Raw) | Shell (Curated/Gold) |
|--------|-------------|----------------------|
| Audit calls | `auditWriter.startRun(...)` | `audit_start_run "..." ` |
| Counting | `df.count()` on persisted DF | `audit_get_count "SELECT COUNT(*)..."` |
| Config | `AuditConfig` case class | Sourced `audit_config.sh` |
| Kill switch | `audit.enabled=false` | `AUDIT_ENABLED=false` |
| Error capture | try/catch, Scala exceptions | `$?` exit code checks |
| App ID | `spark.sparkContext.applicationId` | YARN app ID (if available) or NULL |
| Rule counting | `.agg(sum(when(...)))` | `SUM(CASE WHEN ... END)` in HiveQL |

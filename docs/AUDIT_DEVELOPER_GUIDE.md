# RMS Audit Framework — Developer Guide

This is the working guide for adding auditing to the RMS pipelines (Raw, Curated, Gold).
Read this once before running any of the prompts in `prompts/`. It explains **what we are
building, the rules that keep it safe, and the order to build it in**.

---

## 1. What we are building (in one paragraph)

Today, when a load fails, support digs through Spark logs and runs manual Hive counts.
We are adding a set of **shared audit tables** that every layer (Raw, Curated, Gold) writes to,
so that one set of queries can answer: *Did the job run? Did every file/table load? Do the
counts tie out end to end? What failed, and can I safely rerun it?*

We do **not** change any business logic, transformations, or existing table schemas in Raw.
(Curated and Gold get two small tracking columns — see section 6.)

---

## 2. The data flow (for context)

```
MQ  →  HDFS Sequence Files  →  XMLToHive  →  13 Raw tables
                                                  |
                                   (1-to-1)  Curated tables
                                                  |
                                (1-to-many)  Gold tables
```

- Raw → Curated is one-to-one (each Curated table reads specific Raw tables).
- Curated → Gold fans out (one Curated table can feed multiple Gold tables).

---

## 3. The audit tables (one shared set for all layers)

All three layers write to the **same** audit tables. A `layer` column ('RAW', 'CURATED',
'GOLD') says who wrote the row.

| Table | One row per… | Answers |
|---|---|---|
| `audit_run_control` | run status change | Did the job run? When? Success or failure? |
| `audit_source_control` | input unit (a file for Raw; a source table + window for Curated/Gold) | Was every input fully processed? Input = parsed + rejected? |
| `audit_stage_summary` | target table write | Did each of the 13 (or N) tables load? Expected vs actual rows? |
| `audit_rule_result` | business / DQ / referential-integrity rule | How many rows passed/failed each rule? |
| `audit_merge_summary` | CDC / merge / SCD operation | How many inserts / updates / deletes / expirations? |
| `audit_reconciliation` | entity, per layer hop | Raw = Curated + rejected + filtered? Curated vs Gold? |
| `audit_lineage` | batch-to-batch edge | Which Raw batches fed which Curated batch fed which Gold batch? |
| `audit_error_detail` | individual error | What exactly failed, in which file/message/table? |

---

## 4. The five rules (do not break these)

**Rule 1 — Append-only. Never update.**
Hive on our stack can't update rows. Every status change is a **new row**. "Current status"
means *the latest row for that run_id/batch_id* (`max(created_ts)`). All queries and
dashboards read latest-row-wins. Never write Hive `UPDATE` statements.

**Rule 2 — Audit failures never hide business failures.**
If writing an audit row throws, log it and re-throw the **original** business exception.
Never let an audit problem mask the real problem, and never let an audit failure trigger
more audit writes (no recursion).

**Rule 3 — Counts are Spark jobs. Count once.**
Every `.count()` re-runs computation. So: persist a DataFrame before counting it, compute
**all rule counts in a single aggregation pass** (`sum(when(...))` per rule — not one
filter+count per rule), and count written rows from staging output, never by re-scanning
the final table.

**Rule 4 — No PHI in audit tables by default.**
This is membership/Medicare data. `record_payload` capture is **off by default**, and when
enabled it is truncated and masked per config. Sample failed keys are capped (default 20).

**Rule 5 — There is a kill switch.**
`audit.enabled=false` must make every pipeline behave exactly as it does today. That is our
rollback plan.

---

## 5. How each layer uses the tables

### Raw (XMLToHive)
- One `run_control` row-pair (STARTED / COMPLETED-or-FAILED) per execution.
- One `source_control` row-pair per Sequence File, with
  `input_message_count = parsed + rejected` enforced.
- Each of the 13 DataFrames is written to a **staging path** first
  (`.../staging/<run_id>/<batch_id>/<table>/`), counted there, and only committed to the
  final Raw table if the staged count matches the expected count. That staged count is the
  `actual_written_count` — we trust the final commit; we cannot re-verify rows inside the
  Raw table because its schema is frozen.
- Rerun protection: before processing a file, check `audit_source_control` for a COMPLETED
  row with the same file identity (path + name + size + mod time + checksum). Skip unless
  force-rerun. On retry of a PARTIAL file, skip tables whose stage rows are already
  COMPLETED for the prior batch.

### Curated
- Same run/source pattern. `source_control` records each Raw table read, plus the
  watermark / CDC window used.
- Business rules and DQ checks write to `audit_rule_result` (one aggregation pass).
- CDC results (insert/update/delete/no-op/late/out-of-order) go to `audit_merge_summary`
  — shaped to however the code *actually* applies changes (analyze first; our Spark
  version has no `MERGE INTO`).
- Reconciliation row per entity: `raw_count = curated_count + rejected + filtered`, with a
  named reason for every difference. An unexplained difference fails the run.

### Gold
- Same run/source pattern; `source_control` records Curated inputs + processing window.
- Entity counts per Gold table → `stage_summary`. SCD1/SCD2/PIT activity → `merge_summary`.
  Referential-integrity checks → `audit_rule_result` (rule_type = 'RI').
- Reconciliation per entity across Curated → Gold, with expected fan-out reasons.
- **Lineage is batch-level, not row-level.** `audit_lineage` stores which source batches
  produced which target batches. Row-level tracing works by joining on the `_audit_run_id`
  / `_audit_batch_id` columns carried in Curated and Gold rows plus business keys
  (transaction header → member, etc.). We never build a row-per-record lineage table.

---

## 6. What changes where

| Layer | Schema changes | Code changes |
|---|---|---|
| Raw tables | **None.** Frozen. | XMLToHive gains audit calls + staging write path |
| Curated tables | Add `_audit_run_id`, `_audit_batch_id` columns | Audit calls, rule counting, reconciliation |
| Gold tables | Add `_audit_run_id`, `_audit_batch_id` columns | Audit calls, RI checks, lineage, reconciliation |
| New | 8 audit tables in a dedicated audit database | `AuditWriter` + models, shared by all layers |

---

## 7. What we deliberately do NOT audit

- **Per-transformation counts** (trim, uppercase, defaults) — input always equals output;
  pure noise. We audit at the stage level, and separately audit rules that reject rows.
- **Every join** — only the two or three load-bearing joins get matched/unmatched counts.
- **Performance metrics** (shuffle, memory, executors) — Spark event logs already have
  them. We store `spark_application_id` in `run_control` so you can jump straight from an
  audit row to the Spark UI. That's the link between audit and logs.
- **Row-level lineage tables** — see section 5 / Gold. Batch lineage + carried keys.

---

## 8. Build order (phases)

Each phase is useful on its own. Do not start a later phase before the earlier one works.

1. **Phase 0 — Foundation**: unified DDL, Scala models, config, `AuditWriter`. (Prompts 01–04)
2. **Phase 1 — Raw**: instrument XMLToHive: run/file audit, staging + count validation,
   error audit, rerun protection. (Prompts 05–08)
3. **Phase 2 — Curated**: analyze, instrument, rules, CDC audit, Raw↔Curated
   reconciliation. (Prompts 09–13)
4. **Phase 3 — Gold**: analyze, instrument, RI/SCD audit, lineage, end-to-end
   reconciliation. (Prompts 14–18)
5. **Phase 4 — Wrap-up**: operational query pack, tests, final review. (Prompts 19–21)

---

## 9. Day-to-day: how support will use this

```sql
-- Did last night's Raw load work?          → latest run_control rows for layer='RAW'
-- Which file failed?                        → source_control where latest status != 'COMPLETED'
-- Which of the 13 tables didn't load?       → stage_summary for that batch_id
-- Why were rows rejected?                   → error_detail for that batch_id
-- Do Raw/Curated/Gold counts tie out?       → audit_reconciliation for the run date
-- Where did this Gold record come from?     → its _audit_batch_id → audit_lineage → source batches → source file
-- Is it safe to rerun file X?               → source_control history for that file identity
```

The full query pack lives with the DDL (built in prompt 19).

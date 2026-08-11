# RMS Audit — v2 prompt set (remediation + Gold)

This set is run **on top of** work Copilot has already done from the numbered prompts in
`prompts/`. Those originals are **frozen** — they are the record of what was handed over and
must not be edited. Where v2 disagrees with them, **v2 wins**.

---

## 1. What state this set assumes

| | |
|---|---|
| Already built | Phase 0 foundation (01–04, 03b/04b), Phase 1 Raw/Scala audit (05–08), Phase 2 Curated (09–13) |
| Not started | Phase 3 Gold (14–18), Phase 4 wrap-up (19–21) |
| Audit tables | Created in Hive, containing **test data only** — safe to drop and recreate |
| Curated | May or may not have been run against real data — R01 establishes this |

If any of that is wrong, stop and say so before running R02. In particular, if the audit
tables hold data anyone needs, R02 must be reworked as `ALTER TABLE ADD COLUMNS`.

---

## 2. Run order

```
R01  Diagnose current state            READ ONLY — writes docs/AUDIT_V2_BASELINE.md
─────────────────────────────────────────────────────────────────────────────
R02  Fix audit DDL                     ┐
R03  Fix Scala audit models            │  ONE ATOMIC BLOCK
R04  Fix shell audit library           │  do not run any pipeline between these
R05  Fix Curated instrumentation       ┘
─────────────────────────────────────────────────────────────────────────────
R06  Verify Curated fixes              proves the repair against real data
─────────────────────────────────────────────────────────────────────────────
G14  Analyze Gold pipeline             HARD GATE — G15–G18 read its output files
G15  Gold run/source/entity audit
G16  Gold rules, RI and history
G17  Gold lineage
G18  Gold reconciliation and gate
```

### R02–R05 must be applied together

This is one coordinated breaking change. A partial application leaves a system that **looks
like it works**:

- New DDL with the old Scala models → the model-vs-DDL drift test fails the build.
- New shell library signatures with old Curated callers → **silent corruption.** Bash
  arguments are positional. `audit_complete_source` gains `source_name` as its third
  parameter, so an un-updated caller passes `input_count` into the `source_name` column,
  `processed_count` into `input_count`, and so on. Nothing errors. Every audit row is wrong.

Do not run the Curated or Raw pipeline between R02 and R05. Complete the block, then R06.

---

## 3. What v2 supersedes

| v2 prompt | Supersedes / patches |
|---|---|
| R02 | `prompts/02_Create_Unified_Audit_Schema_DDL.md` |
| R03 | `prompts/03_Create_Audit_Models_And_Config.md`, `prompts/04_Create_AuditWriter.md` |
| R04 | `prompts/03b_Shell_Audit_Library.md` |
| R05 | `prompts/10`, `prompts/11`, `prompts/13` (Curated instrumentation) |
| G14 | `prompts/14_Analyze_Gold_Pipeline.md` |
| G15 | `prompts/15_Gold_Run_Source_And_Entity_Audit.md` |
| G16 | `prompts/16_Gold_Rules_RI_And_History_Audit.md` |
| G17 | `prompts/17_Gold_Lineage.md` |
| G18 | `prompts/18_End_To_End_Reconciliation.md` |

Not superseded, still valid: `prompts/12_Curated_CDC_Merge_Audit.md`, and the Raw prompts
05–08.

### `docs/AUDIT_DEVELOPER_GUIDE.md` is frozen and wrong in two places

It is still the best overview of *why* this framework exists, and its five rules still hold.
But two statements in it are now incorrect and must not be followed:

1. **Its data-flow diagram says Curated → Gold is "1-to-many".** It is **many-to-many**: one
   Gold table is assembled from several Curated tables (fan-in) *and* one Curated table feeds
   several Gold tables (fan-out). Read `docs/GOLD_FANOUT_DESIGN.md` instead — the whole Gold
   track depends on getting this right.
2. **Its table list has 8 tables and omits `audit_gold_source_map`**, the reference table
   R02 adds. Section 6's "8 audit tables" should read "8 audit event tables plus
   `audit_gold_source_map`".

---

## 4. The five rules (unchanged, restated because the guide is frozen)

1. **Append-only. Never update.** Hive on this stack cannot update rows. Every status change
   is a new row; "current status" means the latest row for that key by `created_ts`. Never
   write a Hive `UPDATE`.
2. **Audit failures never hide business failures.** If an audit write throws, log it and
   re-throw the original business exception. Never let an audit failure trigger more audit
   writes.
3. **Count once.** Compute all rule counts in a single aggregation pass. Never re-scan a
   target table to produce a number the run already computed.
4. **No PHI in audit tables by default.** Payload capture off unless configured, truncated
   and masked, sample keys capped.
5. **There is a kill switch.** `AUDIT_ENABLED=false` must restore today's exact behaviour.

**A sixth rule this set adds, because breaking it caused a live bug:** never aggregate over
audit rows. `SUM(input_count)` across an append-only table double-counts, because both
`audit_start_source` and `audit_complete_source` write that column. Always reduce to the
latest row per key first, then aggregate across keys if you need to.

---

## 5. Phase 4 debt (19–21, not yet built)

When those prompts are reached they will need changes this set does not make:

- **19 (ops queries)** — reconciliation queries must carry `source_name` / `target_table` and
  read at edge grain. The `v_reconciliation_daily` view is one row per Curated→Gold edge per
  day, not per entity per day; its RAW rollup must deduplicate to the latest row per
  `(batch_id, source_name)` before summing, or a retried file is counted twice.
- **20 (tests)** — superseded by **`docs/AUDIT_TEST_CASES.md`**. `prompts/20` is entirely
  ScalaTest, so it covers Raw only; the shell layers had no coverage at all, which is why
  every defect R05 fixes reached shipped code. The new document covers all three layers in
  three tiers (static checks, Scala unit tests, integration assertions against the test Hive
  database) and maps each known defect to the case that now catches it.
- **21 (final review)** — its checklist says "8 audit tables"; expect nine objects.

Gold must land first. Do not start Phase 4 before G18 is verified.

# RMS Audit Framework — Test Cases

What must be true of the audit framework, at a grain someone can execute and record
pass/fail against.

**Supersedes `prompts/20_Tests.md`**, which stays frozen. That prompt is entirely ScalaTest,
so it covers the Raw layer only. Curated and Gold are shell scripts calling `hivebeeline`,
and had no test coverage at all — which is why the six defects below all shipped.

| Defect found | Layer | Caught by `prompts/20`? | Covered here by |
|---|---|---|---|
| `SUM(input_count)` double-counts append-only rows | Curated shell | No | **S4, IB1, IB2** |
| Positional argument misbind on `audit_complete_source` | Curated shell | No | **S3, IB3** |
| `${raw_run_id}` never selected | Curated shell | No | **IB4** |
| Reconciliation HQL missing two columns | Curated HQL | Partly | **S5, IB5, II1** |
| End-of-run tallies counting all append-only rows | Curated shell | No | **IA2, IB6** |
| Gate queries missing a `layer` filter | Curated shell | No | **S6, IB7** |

---

## How to use this

Three tiers, different costs, run at different times.

| Tier | What | Needs | When |
|---|---|---|---|
| **S** | Static checks on the source | grep | Every change |
| **U** | Scala unit tests | existing ScalaTest setup | Every build |
| **I** | Integration assertions | test Hive database + one controlled run | Before release |

There is no shell test framework, so Tier S cases are written as greps and Tier I cases as
SQL run after a controlled pipeline execution. If `bats` is adopted later, **S3** and the
Tier I-C / I-G cases become proper unit tests with a stubbed `hivebeeline`; nothing here
depends on that happening.

Substitute real names for `${AUDIT_DB}` and the illustrative table names throughout.

---

## Tier S — static checks

Cheapest tier and the highest value per unit of effort: each case catches a whole class of
defect before anything runs.

| ID | Proves | How | Expected |
|---|---|---|---|
| **S1** | Append-only is never violated | `grep -rniE '\bupdate\b[[:space:]]+[a-z_.]*audit_'` over all audit code and HQL | No matches. Hive cannot update rows; a status change is always a new row |
| **S2** | No call to an undefined function | Extract `audit_[a-z_]*` call sites from every script; diff against the functions defined in `audit_functions.sh` | Empty difference |
| **S3** | No positional-argument misbind | For every `audit_*` call site, count arguments; compare against the `local` count in the definition | Every call site matches its definition exactly |
| **S4** | No un-deduped aggregate over audit rows | `grep -rnE 'SUM\(|COUNT\(\*\)|MAX\(' ` against `audit_source_control`, `audit_stage_summary`, `audit_merge_summary`, `audit_reconciliation` | Every hit is either preceded by a `ROW_NUMBER() ... rn=1` dedup, or is a genuine cross-key rollup with a comment saying so |
| **S5** | Writer contract matches the schema | Parameter count of `audit_write_reconciliation` vs column count of `audit_reconciliation` in the DDL; same for the other writers | Counts equal. Currently 15 params / 15 columns |
| **S6** | Gate queries do not read other layers | Every query against an audit table inside a completion gate or end-of-run rollup | All filter on `layer='<expected>'` |
| **S7** | Kill switch cannot be bypassed | Every function in `audit_functions.sh` | First executable line is the `AUDIT_ENABLED` guard returning 0 |
| **S8** | Raw schemas stay frozen | `grep -rn '_audit_run_id\|_audit_batch_id'` in the Raw DDL | No matches. Only Curated and Gold carry the tracking columns |
| **S9** | No unresolved placeholders shipped | `grep -rnE '\$\{(goldTablesFromParams\|hiveDB_gold\|watermark_condition\|BLOCKING_RULES\|FILTERING_RULES\|REJECTING_RULES)\}'` in generated code | No matches — all replaced with real values from `docs/GOLD_ANALYSIS.md` |

**S3 detail.** This is the one static check that earns its keep on its own. Bash arguments
are positional and untyped, so passing `input_count` where `source_name` is expected produces
no error — every row simply lands with values shifted one column. Current signatures:

| Function | Arity |
|---|---|
| `audit_complete_source` | 8 |
| `audit_fail_source` | 4 |
| `audit_partial_source` | 4 |
| `audit_write_reconciliation` | 15 |

---

## Tier U — Scala unit tests (Raw layer)

Existing ScalaTest + Spark setup. Use temp directories and a test database; never require
production Hive locations.

| ID | Proves | Expected |
|---|---|---|
| **U1** | Model ↔ DDL drift fails the build | Test reads the DDL files and compares against the case classes. **Must now include `source_name` and `target_table` and their position** — this test fails today if it exists |
| **U2** | Identifier generation | `run_id` / `batch_id` unique and well-formed |
| **U3** | Append-only semantics | A status transition adds a row; no row is mutated |
| **U4** | Kill switch | `audit.enabled=false` → zero audit writes, pipeline output identical |
| **U5** | Message reconciliation | `input_count = parsed + rejected` enforced per source |
| **U6** | Staging count match | Counts match → commit proceeds |
| **U7** | Staging count mismatch | Mismatch → **nothing commits** |
| **U8** | Commit failure | Staging retained for retry |
| **U9** | PARTIAL file outcome | Recorded as PARTIAL, not FAILED |
| **U10** | Target coverage | All 13 `RawTarget` mappings present |
| **U11** | Rerun protection | Completed file skipped; force-rerun gets a new `attempt_number`; PARTIAL retry skips already-COMPLETED targets |
| **U12** | Error auditing | Batching, buffer cap, truncation honoured |
| **U13** | PHI default | Payload capture OFF by default |
| **U14** | Audit never masks business failure | Original exception preserved when an audit write throws |
| **U15** | Rule counting | One pass produces correct counts for multiple rules; threshold boundary gives WARNED vs FAILED; sample-key cap respected |
| **U16** | Reconciliation classification | MATCHED / EXPLAINED / MISMATCHED assigned correctly; unexplained difference fails the run |
| **U17** | Lineage edges | Expected edges written for a small three-layer fixture |
| **U18** | Business output unchanged | Golden-output test on a fixture file, auditing ON vs OFF — output byte-identical |

---

## Tier I — integration assertions

### The fixture

One controlled dataset, shaped by what has to be provable. It must contain:

- a Gold table with **≥3 Curated inputs** — one DRIVER, one ENRICH, one LOOKUP → fan-in
- a Curated table feeding **≥2 Gold tables** → fan-out
- an SCD2 entity where **one key receives 3 versions** → distinct-key counting
- at least one intentional filter and one rejecting rule → EXPLAINED vs MISMATCHED
- a `depends_on` pair → build ordering

Volumes should be small enough that expected counts can be worked out by hand. Every case
below is satisfiable from this one fixture.

### I-A — Append-only and latest-row semantics

| ID | Proves | How | Expected |
|---|---|---|---|
| **IA1** | Transitions append | Run one source to completion; count its `audit_source_control` rows | ≥2 rows (STARTED, COMPLETED). This is correct, not duplication |
| **IA2** | Retry counts once | Force a source to fail, then rerun to success | Latest-row read returns **one** row, status COMPLETED. It must not count as both failed and completed |
| **IA3** | Nothing is overwritten | Snapshot row count before and after a status change | Count strictly increased; no row modified |

### I-B — Curated regressions

| ID | Proves | How | Expected |
|---|---|---|---|
| **IB1** | Double-count is gone | Reconciliation on a clean batch | `unexplained_difference = 0`, status MATCHED or EXPLAINED |
| **IB2** | Source count is not doubled | Compare `source_count` against the actual row count in the source window | Equal. Roughly 2× means the `SUM` bug is still present somewhere |
| **IB3** | No argument misbind | Inspect `source_name` on every `audit_source_control` row, **including COMPLETED rows** | Holds a table name. **A number here means a call site still uses the old argument order** |
| **IB4** | Lineage carries a run id | Every `audit_lineage` row for the batch | `source_run_id` non-empty |
| **IB5** | Reconciliation HQL matches the schema | Run the reconciliation INSERT | Succeeds; `source_name` and `target_table` both non-null |
| **IB6** | Tallies agree | `audit_run_control` for the run | `completed_sources + failed_sources ≤ total_sources`; `total_sources` = distinct sources actually processed |
| **IB7** | Gate is layer-scoped | Write a GOLD `audit_source_control` row under the same `run_id`, then run the Curated gate | Curated gate result unchanged |
| **IB8** | MISMATCHED actually fails | Force a mismatch (drop rows from the target between load and reconciliation) | Pipeline **exits non-zero**. Logging alone is a failure of this case |
| **IB9** | PARTIAL when input absent | Run Curated for a window where the Raw side has not completed | Target skipped, PARTIAL row with the reason, other tables still processed, run **not** COMPLETED |
| **IB10** | Rerun is safe | Re-run the same batch | Counts still correct; readers take the latest row |

### I-C — Gold fan-in

| ID | Proves | How | Expected |
|---|---|---|---|
| **IC1** | Batch grain is per target | Run the Gold table with 3 mapped inputs | 3 `audit_source_control` rows, **all sharing one `batch_id`** |
| **IC2** | One stage row per batch | Same run | Exactly 1 `audit_stage_summary` row for that batch |
| **IC3** | Batch id is single-valued | `SELECT DISTINCT _audit_batch_id` on rows written to that Gold table this run | Exactly one value |
| **IC4** | Fan-in gate works | Remove one Curated input for the window, re-run | That target skipped and PARTIAL; **other targets still built**; run not COMPLETED |
| **IC5** | All sources closed | After a successful Gold batch | Every source opened for the batch has a COMPLETED row — not just the DRIVER |
| **IC6** | Blocking failure closes all | Force a blocking rule failure on a fan-in target | Every open source row on that batch is FAILED |

### I-D — Gold fan-out

| ID | Proves | How | Expected |
|---|---|---|---|
| **ID1** | Shared source is not duplication | The Curated table feeding 2+ Gold tables | Its batch appears as `source_batch_id` under each target's lineage. Expected, not an error |
| **ID2** | Edges are independent | Reconciliation rows for that Curated table | One row per target, distinguishable by `target_table`, each with its own status and reason |
| **ID3** | No cross-edge summing | Inspect the reconciliation logic and rows | The identity is asserted per edge only. The Curated count is **never** compared against the total across its targets |

### I-E — Gold reconciliation grain

| ID | Proves | How | Expected |
|---|---|---|---|
| **IE1** | DRIVER edges only | Reconciliation rows vs `audit_gold_source_map` | One row per DRIVER edge, **zero** for ENRICH or LOOKUP |
| **IE2** | Non-driver edges are still audited | For every non-DRIVER edge, look for a matching `RI_*` rule in `audit_rule_result` | One RI rule per non-DRIVER edge. **A missing one means that join is completely unaudited** — reconciliation skips it by design and RI was the only cover |
| **IE3** | Keys, not rows | SCD2 entity whose one key got 3 versions | `target_count = 1`. A 3 means rows are being counted |
| **IE4** | Source count is batch-scoped | The fan-out Curated table's edges | Its `source_count` on each edge equals the window count once — not multiplied by the number of targets |

### I-F — Lineage completeness

| ID | Proves | How | Expected |
|---|---|---|---|
| **IF1** | Every mapped source has an edge | Compare distinct `source_name` count in `audit_lineage` per target against active sources in the map | Equal |
| **IF2** | Incomplete lineage fails the run | Delete one lineage edge, run the completion gate | `lineage_incomplete > 0`, run FAILS. "At least one edge exists" must not be enough |
| **IF3** | No row-level lineage | `SHOW TABLES` in the audit database | No per-record lineage table exists |

### I-G — Source map

| ID | Proves | How | Expected |
|---|---|---|---|
| **IG1** | One driver per target | `GROUP BY gold_table` counting `source_role='DRIVER'` | Exactly 1 each, or explicitly flagged as multi-driver in `docs/GOLD_ANALYSIS.md` |
| **IG2** | Helpers return the right sets | Call `audit_gold_sources_for`, `audit_gold_driver_for` for a known target | All active inputs; the driver |
| **IG3** | Empty map fails loudly | Truncate `audit_gold_source_map`, run Gold | Run **fails or warns explicitly**. Silently completing having processed nothing is the worst outcome in the Gold design and must not be possible |
| **IG4** | Build order respected | The `depends_on` pair | Dependency built before the dependent table |
| **IG5** | Map is reference data | `DESCRIBE audit_gold_source_map` | No `layer` column; not treated as append-only |

### I-H — Safety invariants, shell layers

| ID | Proves | How | Expected |
|---|---|---|---|
| **IH1** | Kill switch restores today's behaviour | Run Curated and Gold with `AUDIT_ENABLED=false` | Zero audit rows for that run; business output byte-identical to the ON run |
| **IH2** | Audit never masks business failure | Point the audit config at an unreachable database, then run a batch that fails for a business reason | Pipeline reports the **original** business error and exit code, not an audit error |
| **IH3** | No audit recursion | Same setup as IH2 | An audit write failure triggers no further audit writes |
| **IH4** | No PHI | Inspect `audit_error_detail` after a run with rejected records | `record_payload` null by default; when enabled, truncated to the configured length; `sample_failed_keys` within the cap |

### I-I — Deployed schema

| ID | Proves | How | Expected |
|---|---|---|---|
| **II1** | Reconciliation schema is current | `DESCRIBE FORMATTED audit_reconciliation` vs the DDL file | Columns match exactly, **in order** — the writers insert positionally |
| **II2** | Map table deployed | `DESCRIBE FORMATTED audit_gold_source_map` | Matches the documented columns |
| **II3** | Nothing missing | `SHOW TABLES` | All 8 event tables plus `audit_gold_source_map` |

---

## Mutation checks — proving the tests are real

A regression test nobody has watched fail is not yet a test. After the suite passes,
reintroduce each defect one at a time and confirm the named case goes red.

| Reintroduce | Must fail |
|---|---|
| Change `source_count` back to `SELECT SUM(input_count) ... WHERE run_id AND batch_id` | **IB1**, **IB2**, **S4** |
| Drop the `source_name` argument from one `audit_complete_source` call | **IB3**, **S3** |
| Remove `run_id` from the lineage `SELECT`, pass `${raw_run_id}` | **IB4** |
| Remove `source_name` / `target_table` from the reconciliation INSERT | **IB5**, **S5** |
| Restore `COUNT(DISTINCT source_name)` + bare `COUNT(*)` end-of-run tallies | **IA2**, **IB6** |
| Drop `layer=` from a gate query | **IB7**, **S6** |
| Delete one `audit_lineage` edge | **IF2** |
| Truncate `audit_gold_source_map` | **IG3** |
| Write one Gold source row per batch instead of per target | **IC1**, **IC5** |

If a listed case does **not** fail, the case is not testing what it claims and needs
rewriting before it is trusted.

---

## Coverage notes

**Where the risk actually sits.** IB1–IB3, IC1, IE2 and IG3 cover failure modes that produce
*plausible wrong data rather than errors*. Nothing upstream catches them: the pipeline
succeeds, the counts look reasonable, and the numbers are wrong. They deserve the most
scrutiny in review.

**Not covered here, deliberately.** Performance and runtime of the audit queries — Spark
event logs and `spark_application_id` in `audit_run_control` already give that, and
`docs/AUDIT_DEVELOPER_GUIDE.md` §7 explains why performance metrics are out of scope for the
audit tables. The one exception worth watching by hand is RI check duration in G16, which can
exceed the load it audits.

**Prompt 20 cross-check.** Every case listed in `prompts/20_Tests.md` appears above as U1–U18.
Nothing from it was dropped; U1 was extended for the new reconciliation columns.

# Prompt 19 — Operational query pack

Create one SQL script of support queries for the audit tables. CRITICAL: every query must
use latest-row-wins semantics (latest created_ts per run_id / batch_id / key) — the
tables are append-only event logs, never point-in-time snapshots. Build a reusable
latest-row pattern (or views like v_current_runs, v_current_sources) first, then write
the queries against the views.

Queries (parameterized with placeholders for db, layer, run_id, batch_id, date):
1. Latest runs per layer with current status.
2. Failed / partial / stuck-in-STARTED runs (stuck = no terminal row after N hours).
3. Failed or partial source files/tables for a run.
4. Message reconciliation mismatches (input != processed + rejected).
5. Stage results for one batch: all 13 Raw targets (or N Curated/Gold targets) with
   expected vs actual and validation status.
6. Rule results for a run: failures first, with sample keys.
7. Merge/SCD activity per target per day.
8. Error details for a run/batch, grouped by error_type.
9. Files processed more than once (rerun history via attempt_number / file identity).
10. Reconciliation: run totals vs source totals vs stage totals (internal consistency).
11. The end-to-end entity view from prompt 18 (raw→curated→gold per entity per day).
12. Daily summary: one row per pipeline per day — status, counts, errors, duration.
13. Lineage walk: given a gold batch_id, list contributing curated batches, raw batches,
    and source files.
14. KPI trend: daily entity counts (members, subscribers, coverages, addresses,
    providers) from reconciliation/stage rows, for anomaly eyeballing. (This is a query,
    not a new table.)

Keep everything compatible with our Hive version (no features prompt 01 flagged as
unavailable).

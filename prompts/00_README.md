# RMS AUDIT — PROMPT PACK (Raw + Curated + Gold, unified model)

Run the prompts in numerical order, one at a time. Review each result before the next.
Read `docs/AUDIT_DEVELOPER_GUIDE.md` first — the prompts assume its rules.

Scope and non-negotiables (repeat these to the assistant if it drifts):
- One shared audit schema for all three layers (`layer` column), 8 tables.
- Raw table schemas are FROZEN. Curated/Gold get only `_audit_run_id`, `_audit_batch_id`.
- Audit tables are APPEND-ONLY. Status = latest row per key. No Hive UPDATE, ever.
- Audit failures must never hide business failures.
- `audit.enabled=false` restores today's exact behavior (kill switch).
- No PHI in audit tables by default; payload capture off unless configured, always truncated.
- Match the project's actual Scala / Spark / Hive versions (prompt 01 discovers them —
  do not assume).

Phases:
- Phase 0  Foundation .......... 01–04
- Phase 1  Raw ................. 05–08
- Phase 2  Curated ............. 09–13
- Phase 3  Gold ................ 14–18
- Phase 4  Wrap-up ............. 19–21

Tips:
- Prompts 01, 09, 14 are analyze-only. Do not let the assistant write code during them.
- If a later prompt needs a decision (e.g., staging path root, audit DB name), make the
  decision yourself and state it in the prompt — don't let the assistant guess.

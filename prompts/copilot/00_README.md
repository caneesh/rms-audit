# Copilot Prompts — RMS Audit Framework (Curated & Gold)

Small, self-contained prompts to give to Copilot in sequence.

**Note:** Raw layer is already implemented. These prompts cover Curated and Gold only (shell/beeline).

## How to use

1. **Run prompts in order** (S01, S02, S03, S04, then 09a, 09b, etc.)
2. **Where you see `[PASTE: ...]`**, paste the actual code from your repo
3. **Review Copilot's output** before moving to the next prompt
4. **Save outputs** — later prompts may reference them

## Prompt Sequence

### Step 1: Shell Audit Library (run first)

| Prompt | Purpose | Output |
|--------|---------|--------|
| S01 | Create shell audit config | audit_config.sh |
| S02 | Create shell audit functions (lifecycle) | audit_functions.sh (part 1) |
| S03 | Create shell audit functions (detail) | audit_functions.sh (part 2) |
| S04 | Create HiveQL templates | Rule counting HQL |

### Step 2: Curated Layer

| Prompt | Purpose | Output |
|--------|---------|--------|
| 09a | Analyze Curated scripts | Script understanding |
| 09b | Analyze Curated HQL | Table mappings, rules |
| 10a | Add Curated run audit | Master script changes |
| 10b | Add Curated source audit | Table loop changes |
| 11a | Create Curated rule HQL | audit_rules_curated.hql |
| 13a | Add Curated reconciliation | Recon logic |

### Step 3: Gold Layer

| Prompt | Purpose | Output |
|--------|---------|--------|
| 14a | Analyze Gold scripts | `docs/GOLD_ANALYSIS.md` + `sql/audit_gold_source_map_seed.sql` |
| 15a | Add Gold run/source audit | Gold script changes |
| 18b | Create reconciliation view | v_reconciliation_daily |

**14a is a hard gate.** 15a and 18b read the two files it writes; they do not have its
conversation. Load the seed SQL into `audit_gold_source_map` before running 15a, or its
loops return nothing and the instrumentation silently does no work.

**This track is incomplete for Gold.** There is no short-form prompt for Gold rules and RI
(numbered 16), Gold lineage (17), or the Gold reconciliation and completion gate (18a) —
18b only builds the reporting view. To finish Gold, use the numbered prompts
`prompts/16`, `17` and `18` directly; they assume repo access rather than pasted snippets,
which suits Copilot working inside the pipeline repo.

### Step 4: Operations

| Prompt | Purpose | Output |
|--------|---------|--------|
| 19a | Create operational queries | Query pack |

---

## Key Rules (repeat to Copilot if it drifts)

1. **Append-only** — Audit tables never UPDATE, only INSERT new rows
2. **Kill switch** — `AUDIT_ENABLED=false` must restore today's exact behavior  
3. **No PHI** — Payload capture is OFF by default
4. **Audit failures don't hide business failures** — Log and continue
5. **One-pass counting** — Rules counted in single table scan via UNION ALL

---

## Files created by these prompts

```
scripts/audit/
├── audit_config.sh
├── audit_functions.sh
└── hql/
    ├── audit_rules_template.hql
    ├── audit_ri_checks.hql
    └── audit_sample_keys.hql

sql/
└── v_reconciliation_daily.sql

ops/
└── audit_queries.sql
```

---

## Backup

Raw/Scala prompts moved to `backup_raw/` (not needed since Raw is already implemented).

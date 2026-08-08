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
| 16a | Create Gold rule + RI HQL, SCD merge summary | `audit_rules_gold_${table}.hql` |
| 17a | Add Gold lineage | Lineage INSERT + `docs/LINEAGE.md` |
| 18a | Add Gold reconciliation + completion gate | Recon logic, gate |
| 18b | Create reconciliation view | v_reconciliation_daily |

**14a is a hard gate.** Every Gold prompt after it reads the two files it writes; they do
not have its conversation. Load the seed SQL into `audit_gold_source_map` before running
15a, or the loops in 15a/16a/17a/18a return nothing and the instrumentation silently does
no work.

**Run the Gold prompts in order** — 16a reads batch ids that 15a creates, 17a's lineage is
verified by 18a's gate, and 18b reports on rows 18a writes.

Read `docs/GOLD_FANOUT_DESIGN.md` before starting Gold. Curated→Gold is many-to-many and
the batch grain differs from Curated's: one batch per Gold *target*, with N source rows
sharing it.

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
    ├── audit_rules_gold_${table}.hql
    ├── audit_ri_checks.hql
    └── audit_sample_keys.hql

sql/
├── audit_gold_source_map_seed.sql
└── v_reconciliation_daily.sql

docs/
├── GOLD_ANALYSIS.md
└── LINEAGE.md

ops/
└── audit_queries.sql
```

---

## Backup

Raw/Scala prompts moved to `backup_raw/` (not needed since Raw is already implemented).

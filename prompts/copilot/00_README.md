# Copilot Prompts — RMS Audit Framework

Small, self-contained prompts to give to Copilot in sequence.

## How to use

1. **Run prompts in order** (01a, 01b, 01c, then 02a, 02b, etc.)
2. **Where you see `[PASTE: ...]`**, paste the actual code from your repo
3. **Review Copilot's output** before moving to the next prompt
4. **Save outputs** — later prompts may reference them

## Architecture

| Layer | Technology | Audit mechanism |
|-------|------------|-----------------|
| Raw | Scala/Spark | AuditWriter class (prompts 03-04) |
| Curated | Shell + beeline | audit_functions.sh (prompts S01-S04) |
| Gold | Shell + beeline | audit_functions.sh (prompts S01-S04) |

All layers write to the same audit tables (prompt 02).

---

## Prompt Sequence

### Phase 0: Foundation

| Prompt | Purpose | Output |
|--------|---------|--------|
| 01a | Analyze Raw entry point | Flow understanding |
| 01b | Analyze Raw data flow | 13 table mappings |
| 01c | Analyze Raw error handling | Insertion points |
| 02a | Create audit DDL (control tables) | 2 table DDLs |
| 02b | Create audit DDL (detail tables) | 6 table DDLs |
| 03a | Create Scala case classes | 8 model classes |
| 03b | Create Scala AuditConfig | Config class |
| 04a | Create AuditWriter (lifecycle) | Run/source methods |
| 04b | Create AuditWriter (detail) | Stage/error methods |

### Phase 0-Shell: Shell audit library

| Prompt | Purpose | Output |
|--------|---------|--------|
| S01 | Create shell audit config | audit_config.sh |
| S02 | Create shell audit functions (lifecycle) | audit_functions.sh (part 1) |
| S03 | Create shell audit functions (detail) | audit_functions.sh (part 2) |
| S04 | Create HiveQL templates | Rule counting HQL |

### Phase 1: Raw (Scala)

| Prompt | Purpose | Output |
|--------|---------|--------|
| 05a | Instrument Raw run lifecycle | Main class changes |
| 05b | Instrument Raw file processing | File loop changes |
| 06a | Create staging write function | writeWithAudit() |

### Phase 2: Curated (Shell)

| Prompt | Purpose | Output |
|--------|---------|--------|
| 09a | Analyze Curated scripts | Script understanding |
| 09b | Analyze Curated HQL | Table mappings, rules |
| 10a | Add Curated run audit | Master script changes |
| 10b | Add Curated source audit | Table loop changes |
| 11a | Create Curated rule HQL | audit_rules_curated.hql |
| 13a | Add Curated reconciliation | Recon logic |

### Phase 3: Gold (Shell)

| Prompt | Purpose | Output |
|--------|---------|--------|
| 14a | Analyze Gold scripts | Script understanding |
| 15a | Add Gold run/source audit | Gold script changes |
| 18b | Create reconciliation view | v_reconciliation_daily |

### Phase 4: Wrap-up

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
6. **Frozen Raw schemas** — Raw tables cannot have audit columns added

---

## Files created by these prompts

```
# Scala (Raw layer)
src/main/scala/com/yourcompany/audit/
├── model/
│   ├── AuditRunControl.scala
│   ├── AuditSourceControl.scala
│   ├── AuditStageSummary.scala
│   ├── AuditRuleResult.scala
│   ├── AuditMergeSummary.scala
│   ├── AuditReconciliation.scala
│   ├── AuditLineage.scala
│   └── AuditErrorDetail.scala
├── AuditConfig.scala
└── AuditWriter.scala

# Shell (Curated/Gold layers)
scripts/audit/
├── audit_config.sh
├── audit_functions.sh
└── hql/
    ├── audit_rules_template.hql
    ├── audit_ri_checks.hql
    └── audit_sample_keys.hql

# DDL
sql/
├── audit_run_control.sql
├── audit_source_control.sql
├── audit_stage_summary.sql
├── audit_rule_result.sql
├── audit_merge_summary.sql
├── audit_reconciliation.sql
├── audit_lineage.sql
├── audit_error_detail.sql
└── v_reconciliation_daily.sql

# Operations
ops/
└── audit_queries.sql
```

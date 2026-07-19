# Prompt 11 — Curated business-rule and data-quality audit

Wire the rules found in prompt 09 into audit_rule_result. Do NOT change what any rule
does — only count and record it.

Rules to cover (from the prompt-09 inventory; extend with any missed):
- BUSINESS: inactive members removed, invalid products rejected, missing corp codes,
  future effective dates, duplicate coverage, bad enrollments, ...
- DQ: null MID, null corp, invalid DOB / future DOB, invalid gender, invalid state,
  missing coverage, ...
- Duplicates: duplicate business keys, duplicate source records, duplicate CDC events —
  record as DQ rules with rule_name prefixed 'DUP_'.

## The one-pass rule (mandatory)
Compute ALL rule counts for a target in a SINGLE aggregation over the persisted
DataFrame: one .agg() with sum(when(ruleViolated, 1).otherwise(0)) per rule.
Do NOT emit one filter().count() job per rule. Sample failed keys: collect at most
AuditConfig.sampleKeyCap keys per rule (a single additional bounded job per target is
acceptable for samples; skip samples for rules with zero failures).

Write all rule rows for a target with one writeRuleResults(batch) call.

## Join audit (selective)
Only for the load-bearing joins identified in prompt 09: record left/right/matched/
unmatched counts as rule_result rows (rule_type='DQ', rule_name='JOIN_<name>'), computed
within the same pass where feasible. Do not instrument every join.

## Rule outcome policy
Each rule carries threshold_pct from config (default 0 = any failure fails).
failed_count within threshold → WARNED; above → FAILED; a FAILED blocking rule fails the
target's stage. Mark which rules are blocking vs warning in config, not in code.

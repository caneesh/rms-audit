# Prompt 03a — Create Scala audit case classes

Create Scala case classes matching the audit DDL from prompts 02a/02b.

**Scala version:** [PASTE: version from 01a]
**Spark version:** [PASTE: version from 01a]

**Rules:**
- java.sql.Timestamp for TIMESTAMP columns
- Long for BIGINT, Int for INT, Double for DOUBLE
- Use Option[] only where Spark encoders handle it safely for your version
- Add companion object with factory methods for status transitions (e.g., `started()`, `completed()`, `failed()`)

Create these case classes:

```scala
// Example structure — adjust types for your Spark version
case class AuditRunControl(
  layer: String,
  runId: String,
  pipelineName: String,
  // ... rest of columns from audit_run_control
)

object AuditRunControl {
  def started(layer: String, runId: String, ...): AuditRunControl = ???
  def completed(base: AuditRunControl, ...): AuditRunControl = ???
  def failed(base: AuditRunControl, errorMessage: String): AuditRunControl = ???
}
```

Create case classes for:
1. AuditRunControl
2. AuditSourceControl
3. AuditStageSummary
4. AuditRuleResult
5. AuditMergeSummary
6. AuditReconciliation
7. AuditLineage
8. AuditErrorDetail

Put in package: `com.yourcompany.audit.model` (adjust as needed)

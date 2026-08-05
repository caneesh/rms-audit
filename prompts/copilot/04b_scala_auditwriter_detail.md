# Prompt 04b — Create AuditWriter (stage/error/rule methods)

Continue the AuditWriter class with methods for stage summary, rules, merge, reconciliation, lineage, and errors.

**Add these methods to AuditWriter:**

```scala
// ============ STAGE SUMMARY ============

def writeStageSummary(
  layer: String,
  runId: String,
  batchId: String,
  sourceName: String,
  stageName: String,
  targetTable: String,
  inputCount: Long,
  expectedOutputCount: Long,
  actualWrittenCount: Long,
  rejectCount: Long,
  status: String,  // COMPLETED, FAILED
  startTime: Timestamp,
  endTime: Timestamp,
  errorMessage: Option[String] = None
): Unit = ???

// ============ RULE RESULTS ============

// Write a batch of rule results (more efficient than one at a time)
def writeRuleResults(rules: Seq[AuditRuleResult]): Unit = {
  if (!auditEnabled || rules.isEmpty) return
  try {
    rules.toDF()
      .write
      .mode(SaveMode.Append)
      .insertInto(s"${config.auditDatabase}.audit_rule_result")
  } catch {
    case e: Exception => log.error(s"Failed to write rule results: ${e.getMessage}", e)
  }
}

// ============ MERGE SUMMARY ============

def writeMergeSummary(summary: AuditMergeSummary): Unit = ???

// ============ RECONCILIATION ============

def writeReconciliation(recon: AuditReconciliation): Unit = ???

// ============ LINEAGE ============

def writeLineage(
  layer: String,
  runId: String,
  batchId: String,
  targetTable: String,
  sourceLayer: String,
  sourceRunId: String,
  sourceBatchId: String,
  sourceName: String
): Unit = ???

// ============ ERRORS ============

// Buffer errors and write in batches to avoid many small writes
private val errorBuffer = new java.util.concurrent.ConcurrentLinkedQueue[AuditErrorDetail]()

def writeError(error: AuditErrorDetail): Unit = {
  if (!auditEnabled) return
  errorBuffer.add(truncateError(error))
  if (errorBuffer.size() >= config.errorBufferMax) {
    flushErrors()
  }
}

def flushErrors(): Unit = {
  if (!auditEnabled) return
  val errors = new java.util.ArrayList[AuditErrorDetail]()
  var error = errorBuffer.poll()
  while (error != null) {
    errors.add(error)
    error = errorBuffer.poll()
  }
  if (!errors.isEmpty) {
    try {
      import scala.collection.JavaConverters._
      errors.asScala.toSeq.toDF()
        .write
        .mode(SaveMode.Append)
        .insertInto(s"${config.auditDatabase}.audit_error_detail")
    } catch {
      case e: Exception => log.error(s"Failed to flush errors: ${e.getMessage}", e)
    }
  }
}

// Truncate error message and payload per config
private def truncateError(error: AuditErrorDetail): AuditErrorDetail = {
  error.copy(
    errorMessage = error.errorMessage.take(config.errorMessageMaxLength),
    recordPayload = if (config.payloadCaptureEnabled) 
      error.recordPayload.map(_.take(config.payloadMaxLength)) 
    else None
  )
}
```

Implement all methods. Remember:
- Batch writes where possible
- Truncate strings per config
- Log errors but don't throw
- Check `auditEnabled` at method entry

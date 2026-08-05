# Prompt 04a — Create AuditWriter (run/source lifecycle)

Create the AuditWriter class that handles all audit table writes. This prompt covers run and source lifecycle methods.

**Spark version:** [PASTE: version from 01a]

**Key rules:**
1. If `config.enabled = false`, every method is a no-op (return immediately)
2. Append-only: every call INSERTs new rows, never UPDATE
3. Audit failures must NEVER hide business failures — log and continue
4. Never write an audit row about a failed audit write (no recursion)

```scala
class AuditWriter(spark: SparkSession, config: AuditConfig) {
  
  import spark.implicits._
  
  // Check enabled once, use throughout
  private val auditEnabled: Boolean = config.enabled
  
  // ============ RUN LIFECYCLE ============
  
  def startRun(
    layer: String,
    runId: String,
    pipelineName: String,
    processName: String,
    sourceSystem: String,
    sparkAppId: String
  ): Unit = {
    if (!auditEnabled) return
    // Create AuditRunControl.started(...) and write to audit_run_control
    ???
  }
  
  def completeRun(
    runId: String,
    totalSources: Long,
    completedSources: Long,
    failedSources: Long,
    totalInput: Long,
    totalProcessed: Long,
    totalRejected: Long
  ): Unit = {
    if (!auditEnabled) return
    // Create AuditRunControl.completed(...) and write
    ???
  }
  
  def failRun(runId: String, errorMessage: String): Unit = {
    if (!auditEnabled) return
    // Create AuditRunControl.failed(...) and write
    ???
  }
  
  // ============ SOURCE LIFECYCLE ============
  
  def startSource(
    layer: String,
    runId: String,
    batchId: String,
    sourceType: String,  // "FILE" or "TABLE"
    sourceName: String,
    sourcePath: String,
    fileSize: Long,
    fileChecksum: String,
    fileModifiedTime: Timestamp
  ): Unit = ???
  
  def completeSource(
    runId: String,
    batchId: String,
    inputCount: Long,
    processedCount: Long,
    rejectedCount: Long,
    completedTargets: Int,
    failedTargets: Int
  ): Unit = ???
  
  def failSource(runId: String, batchId: String, errorMessage: String): Unit = ???
  
  def markSourcePartial(runId: String, batchId: String, completedTargets: Int, failedTargets: Int): Unit = ???
  
  // ============ PRIVATE HELPERS ============
  
  private def writeToTable[T: Encoder](tableName: String, record: T): Unit = {
    try {
      Seq(record).toDF()
        .write
        .mode(SaveMode.Append)
        .insertInto(s"${config.auditDatabase}.$tableName")
    } catch {
      case e: Exception =>
        // Log error but DO NOT throw — audit failure must not hide business failure
        log.error(s"Failed to write audit record to $tableName: ${e.getMessage}", e)
    }
  }
  
  private def currentTimestamp(): Timestamp = new Timestamp(System.currentTimeMillis())
}
```

Implement all the lifecycle methods. Show explicit column selection matching Hive table order.

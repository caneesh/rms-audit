# Prompt 06a — Create Raw staging write function

Create a reusable function that writes DataFrames with staging validation.

**Context:**
- Raw table schemas are FROZEN — we cannot add audit columns to them
- Validation must happen in staging before commit to final table
- All 13 Raw tables use this same pattern

**Design:**
1. Write to staging path first
2. Count rows in staging
3. If count matches expected, commit to final table
4. If mismatch, fail and retain staging for debugging

```scala
case class RawTarget(
  name: String,
  df: () => DataFrame,  // Lazy to avoid computing before needed
  finalTable: String,
  stagingPath: String => String  // Function of batchId
)

def writeWithAudit(
  target: RawTarget,
  runId: String,
  batchId: String,
  auditWriter: AuditWriter,
  config: AuditConfig
)(implicit spark: SparkSession): Boolean = {
  
  val stageStart = System.currentTimeMillis()
  val stagingPath = target.stagingPath(batchId)
  
  try {
    // 1. Persist the DataFrame (count once, write once)
    val df = target.df().persist(StorageLevel.MEMORY_AND_DISK)
    val expectedCount = df.count()
    
    // 2. Write to staging
    df.write
      .mode(SaveMode.Overwrite)
      .format("parquet")  // or match final table format
      .save(stagingPath)
    
    // 3. Read back and count
    val actualCount = spark.read.parquet(stagingPath).count()
    
    // 4. Validate
    if (expectedCount != actualCount) {
      auditWriter.writeStageSummary(
        layer = "RAW", runId, batchId, target.name, "WRITE", target.finalTable,
        expectedCount, expectedCount, actualCount, rejectCount = 0,
        status = "FAILED", /* times */, errorMessage = Some("Count mismatch")
      )
      auditWriter.writeError(/* COUNT_MISMATCH error */)
      df.unpersist()
      return false  // Do NOT commit
    }
    
    // 5. Commit to final table (preserve existing write behavior)
    spark.read.parquet(stagingPath)
      .write
      .mode(SaveMode.Append)  // or whatever the original mode was
      .insertInto(target.finalTable)
    
    // 6. Write success audit
    auditWriter.writeStageSummary(
      layer = "RAW", runId, batchId, target.name, "WRITE", target.finalTable,
      expectedCount, expectedCount, actualCount, rejectCount = 0,
      status = "COMPLETED", /* times */, errorMessage = None
    )
    
    // 7. Cleanup staging (if policy says so)
    if (config.stagingRetentionPolicy == "delete-on-success") {
      // Delete staging path
    }
    
    df.unpersist()
    true
    
  } catch {
    case e: Exception =>
      auditWriter.writeStageSummary(/* FAILED with error */)
      auditWriter.writeError(/* COMMIT_FAILURE error */)
      false
  }
}
```

Create this function and show how it will be called.

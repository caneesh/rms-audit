# Prompt 05b — Instrument Raw file processing

Add audit instrumentation for each Sequence File processed.

**Context:**
- AuditWriter and runId are set up (from 05a)
- Each Sequence File is a "source" in audit terms
- We need to track: file name, size, message counts, success/failure

**Find the file processing loop and add:**

```scala
// Before processing each file:
val batchId = s"${runId}_${fileName}_${System.currentTimeMillis()}"
val fileSize = // get file size from HDFS
val fileModTime = // get modification time

auditWriter.startSource(
  layer = "RAW",
  runId = runId,
  batchId = batchId,
  sourceType = "FILE",
  sourceName = fileName,
  sourcePath = filePath,
  fileSize = fileSize,
  fileChecksum = "",  // or compute if cheap
  fileModifiedTime = new Timestamp(fileModTime)
)

// Use accumulators for counts (do NOT add a second pass over the file):
val inputCounter = spark.sparkContext.longAccumulator("inputCount")
val processedCounter = spark.sparkContext.longAccumulator("processedCount")
val rejectedCounter = spark.sparkContext.longAccumulator("rejectedCount")

// Inside the existing message processing loop, increment counters:
// inputCounter.add(1)  // for each message read
// processedCounter.add(1)  // for each message successfully parsed
// rejectedCounter.add(1)  // for each message that failed to parse

// After processing the file:
val inputCount = inputCounter.value
val processedCount = processedCounter.value
val rejectedCount = rejectedCounter.value

// Reconciliation check:
if (inputCount != processedCount + rejectedCount) {
  auditWriter.writeError(AuditErrorDetail(
    layer = "RAW",
    runId = runId,
    batchId = batchId,
    errorType = "RECONCILIATION",
    errorMessage = s"Count mismatch: input=$inputCount, processed=$processedCount, rejected=$rejectedCount"
    // ... other fields
  ))
  auditWriter.failSource(runId, batchId, "Count reconciliation failed")
} else {
  auditWriter.completeSource(runId, batchId, inputCount, processedCount, rejectedCount, 
    completedTargets = 13, failedTargets = 0)
}
```

Show the diff of changes to the file processing code.

[PASTE: Your file processing loop code here]

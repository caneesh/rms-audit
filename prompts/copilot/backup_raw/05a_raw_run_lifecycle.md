# Prompt 05a — Instrument Raw run lifecycle

Add audit instrumentation to the XMLToHive Scala application for run-level tracking.

**Context:**
- AuditWriter class exists (from prompts 04a/04b)
- AuditConfig class exists (from prompt 03b)
- This is the Raw layer (layer = "RAW")

**Rules:**
- If `config.auditEnabled = false`, behavior must be identical to today
- Do NOT change any XML parsing, transformations, or write logic
- Audit failures must not stop the pipeline — log and continue

**Find the main entry point and add:**

```scala
// At the start of main() or run() method:
val auditConfig = AuditConfig.load(config)  // Load from existing config
val auditWriter = new AuditWriter(spark, auditConfig)

val runId = java.util.UUID.randomUUID().toString
val sparkAppId = spark.sparkContext.applicationId

auditWriter.startRun(
  layer = "RAW",
  runId = runId,
  pipelineName = "XMLToHive",
  processName = "RawLoad",
  sourceSystem = "MQ",
  sparkAppId = sparkAppId
)

// Store runId for use throughout the job
// ... existing processing code ...

// At the end (in a finally block or after all processing):
auditWriter.completeRun(
  runId = runId,
  totalSources = totalFileCount,
  completedSources = successfulFileCount,
  failedSources = failedFileCount,
  totalInput = totalMessageCount,
  totalProcessed = processedMessageCount,
  totalRejected = rejectedMessageCount
)

// OR if the job fails:
auditWriter.failRun(runId, errorMessage)
```

Show the diff of changes to the main entry point file.

[PASTE: Your main entry point file here]

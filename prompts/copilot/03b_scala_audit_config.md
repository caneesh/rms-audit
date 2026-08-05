# Prompt 03b — Create Scala AuditConfig

Create an AuditConfig case class that integrates with the existing config mechanism.

**Existing config mechanism:** [PASTE: how config is loaded today, e.g., typesafe config, properties]

**Required settings:**

```scala
case class AuditConfig(
  // Master kill switch — false = all audit is no-op
  enabled: Boolean = false,
  
  // Database and paths
  auditDatabase: String = "audit",
  stagingRootPath: String = "/data/staging/audit",
  
  // Staging retention
  stagingRetentionPolicy: String = "delete-on-success", // or "keep-N-days"
  stagingRetentionDays: Int = 7,
  
  // Rerun behavior
  forceRerun: Boolean = false,
  inflightStalenessMinutes: Int = 60,
  
  // Error capture limits
  errorMessageMaxLength: Int = 4000,
  payloadCaptureEnabled: Boolean = false,  // PHI risk — default OFF
  payloadMaxLength: Int = 1000,
  sampleFailedKeysCap: Int = 20,
  errorBufferMax: Int = 10000,
  
  // Rule thresholds (can be per-rule in a Map)
  defaultRuleThresholdPct: Double = 0.0,
  blockingRules: Set[String] = Set.empty
)
```

**Requirements:**
1. Load from the existing config mechanism (show how to wire it in)
2. Provide sensible defaults
3. `enabled = false` must make all audit operations no-op
4. Include a `validate()` method that checks paths exist, database is accessible, etc.

[PASTE: Your existing config loading code or pattern here]

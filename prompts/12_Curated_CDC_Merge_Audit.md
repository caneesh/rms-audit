# Prompt 12 — Curated CDC / merge audit

Implement audit_merge_summary for the Curated apply step, shaped to the ACTUAL mechanism
found in prompt 09 — not to an assumed MERGE INTO.

Depending on the mechanism:
- CDC classification done in Spark before writing (typical for our stack): counts come
  from the classification DataFrame itself — inserted/updated/deleted/noop as a single
  aggregation over the classified set. Cheap; do this wherever the code already labels
  records I/U/D.
- INSERT OVERWRITE of partitions: inserted/updated/deleted cannot be observed directly;
  record what IS knowable (rows written per partition, prior partition counts if already
  computed) and set the unknowable counts to null — do not fabricate them. Say so in the
  response.
- Hive ACID via HiveQL: capture what the mechanism returns; if it returns nothing, fall
  back to before/after aggregate comparison ONLY if that is affordable (partition-scoped);
  otherwise record nulls and note it.

Also record, where the code can see them:
- late_arriving_count (event time older than the current watermark)
- out_of_order_count (event older than the latest applied version for its key)
- duplicate_event_count (same CDC key + version seen more than once in the batch)
These are aggregations over data already in hand — never extra scans of the target table.

One merge_summary row per target table per batch. Failures → error_detail
(error_type='MERGE_FAILURE') and the stage fails.

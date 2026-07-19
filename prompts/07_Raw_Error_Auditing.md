# Prompt 07 — Raw error auditing

Wire audit_error_detail into every Raw failure path:
- Sequence File read failures.
- XML parse failures / malformed messages (these are the 'rejected' messages — every
  rejected message gets an error row, batched).
- DataFrame creation failures.
- Staging write failures, count mismatches, final commit failures.
- Unexpected exceptions (catch-all at file and run level).

Per error row: layer='RAW', run_id, batch_id, source_name, stage_name, target_table
where applicable, error_type, error_code where available, truncated error_message,
error_timestamp.

Message identity: there is no natural message_id in the feed, so set
record_index = the record's ordinal position within the Sequence File, and message_id =
"<file_name>:<record_index>". If prompt 01 found a transaction-header key that is
available at rejection time, use it as message_id instead — state which you used.

Rules:
- record_payload only if audit.payload.enabled=true, truncated and masked per config.
  Default OFF — this is membership/Medicare data (PHI).
- Batch error writes (collect per file, write once) — never one Spark write per error,
  and cap the in-memory error buffer to avoid driver OOM on a poison file (if the cap is
  hit, write the batch and continue buffering).
- The original business exception remains the primary failure — audit writes never
  replace or reorder it.

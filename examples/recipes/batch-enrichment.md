<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Batch enrichment

Enrich an entity list asynchronously and distinguish skipped, failed, fused and database-admitted results.

## Input and settings

Supply entities directly, derive them from existing records, or use `fetch_entities` for a server-side REST GET. The fetch tool unwraps arrays under common wrapper keys and truncates to `max_entities` for response size; `total` is the pre-truncation count. It does not walk source pagination, and the truncation happens after fetching. If the source has more pages, obtain them separately. Treat credentials as credentials, not entity fields.

Read the schema and its `input_contract` before constructing each entity. Use the published document for a linked schema. All preserve paths and keys for supplied array items are required; identifying names are guidance. Call `start_batch_enrichment` with exactly one of `schema_id` or `target_schema`. Omitted models use the task's automatic single-model selection; explicit multiple models enable per-entity fusion when all succeed.

There is no fixed 100-entity cap. Live organization prompt quotas and credits are checked as work advances, including concurrent consumption. Exhaustion can skip the remaining entities. Batches incur normal model costs. Attachments are applied to **every** entity: they are not paired with individual entries. The MCP batch tool exposes neither web-search activation nor `database_sync=false`. For per-entity source lists or opt-out, use individual calls or an appropriate REST/UI flow.

## Start, poll and inspect

`start_batch_enrichment` returns `job_id` and `total`. Poll `get_job_status` using the entity counters: on batch jobs, `total_models` means models per entity and `completed_models` is not maintained. Batch classification never pauses. A confident mismatch skips that entity with `classification_mismatch`; softer verdicts become prompt context. One entity's failure does not imply all others failed.

Terminal summaries include entity outcomes and `db_saved`, `db_partial`, `db_rejected` counts. Use `include_result=true` for per-entity details and persisted IDs. Fetch records through `list_records(job_id=...)`, paging through all results, and use `get_record` selectively. One entity may have multiple model records and an arbitration record; a count of records is not a count of enriched entities. An entity skipped before persistence may appear only in the terminal outcome.

An unknown job is unavailable from the in-memory manager, not proof of success. Check persisted records for that ID. `cancel_job` stops remaining work cooperatively; in-flight calls and their records may complete, and earlier database writes remain.

## Partial failures and delivery

Automatic fusion and database admission require all selected models of an entity to succeed. If one fails, the surviving record is not a fused result. Recover failed expertise domains on their own record with `retry_expertises`. If a leg left no record, run only that missing model through `enrich_entity(database_sync=false)` and manually `merge_records` with the survivor. See [Enrichment and fusion](enrichment-and-fusion.md).

Database counts describe admission, not proof of replica application. Report partial/rejected outcomes and inspect the affected record's delivery state. A zero pending count does not exclude quarantined changes. The current server-side rows are available through `list_entity_states`; external delivery is covered in [Database sync](database-sync.md).

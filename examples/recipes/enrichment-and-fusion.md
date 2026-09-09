<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Enrichment, fusion and recovery

Enrich an entity, interpret its actual outcome and recover partial model failures without repeating successful work.

## Prepare a run

Choose a saved schema with `list_schemas`, then read `get_schema`. For a database-linked schema read its published version; for an unlinked draft use the working copy. Follow `input_contract`: caller-owned preserve values and keys of supplied array items are required. Identifying field names are guidance, not the only accepted input names. Never fabricate a preserve value; if it should be researched, correct the schema with authorization instead.

Omit `models` or use `["auto"]` for the organization's task default. Auto chooses **one** model, so it does not fuse. Omit strategy or use `auto` to choose single-pass, expert-domain or multi-expertise execution from the schema. Multiple explicit models require valid `provider::model` keys. Use `list_models` when selecting these or inspecting account limits; its full catalogue can be large and need not be fetched for every default run. Auto defaults may come from a pinned choice or benchmark scores and account for attachment capabilities.

Pass `languages` for the output: the first language is used for ordinary text, and fields marked multilingual receive all requested languages. Sample language, schema-description language and enrichment languages are separate choices. Attachments supply additional source material. This MCP tool does not expose web-search activation; do not infer research from a model's advertised search capability.

Generation, classification, expertise retries, semantic resolution and optional LLM arbitration can incur costs. Multiple models/domains and benchmark repetitions can multiply work; an API call is not necessarily one provider call. Model availability means a usable key exists, not current provider quota. Plan and credit failures carry details to explain or reduce the request; do not repeatedly retry exhausted account credits.

## Interpret success

`enrich_entity` is synchronous up to `timeout_seconds`. A timeout cancels the job but an in-flight leg may still persist a partial record; inspect its `job_id`. For long runs use `start_batch_enrichment`, including a one-entity batch when its parameter differences are acceptable.

An optional classifier can return `success=false`, `error_code="classification_warning"` and a `classification` object. Relay its reasoning and confidence. Re-call with `force_after_classification_warning=true` only after approval to bypass it. The retry is a new call without classification, not a continuation token.

With two or more models, automatic fusion and entity-layer admission wait for all models to succeed. Inspect `failed_models`: `record_id` can identify a surviving model, not a fusion. The `fusion` block reports the actual method and applied arbiter; passing `arbitration_model` does not prove that it succeeded. Rule-based fallback can still produce a result.

Schema compliance and agreement between models are not proof of factual truth or freshness. Report validation failures, declared unknowns, sources actually used and unresolved disagreements. Consult `get_record` for full output, `failed_expertises`, `partial`, actual fusion sources and costs.

## Recover a failed model

1. Read `list_records(job_id=...)` and inspect the failed model's record with `get_record`.
2. If that record has failed expertise domains, call `retry_expertises` on **that record**, using its `entity_input_data` and `saved_schema_id`. Poll the job, then read the record again. Retrying updates the existing record; records are not universally immutable.
3. If the failed leg persisted no record, retry only that model (or an agreed substitute) with `enrich_entity(database_sync=false)`, then `merge_records` with the surviving and recovered record IDs. Calling `retry_expertises` on the successful sibling returns `no_failed_expertises`.
4. Retry a transient failure on the same model; consider a substitute for a reproducible schema/capability failure. Report the actual source models if a substitution changes them. An explicit-model compatibility failure in an asynchronous start may justify one retry with `auto`; do not cycle blindly through models.

A manual merge can enter the entity layer even when the original multi-model job failed its all-models gate. Disabling sync on the recovery leg avoids an intermediate write before the merge. If keeping only a surviving unfused result instead, state that the failed multi-model run did not write it to the linked database.

## Jobs and database outcomes

Poll asynchronous jobs with `get_job_status`. Running is not failure; paused jobs need `answer_job_question`. Terminal summaries include persisted record IDs and batch database outcome counts. Request `include_result=true` only for detailed outcomes, then fetch selected records. Pagination and `records_truncated` matter on large runs. Unknown jobs may have expired, been lost on restart or never existed: search persisted records without assuming completion. Cancellation does not undo records or database writes.

Distinguish generated output, entity-layer admission and replica delivery. Check `database.status` (`saved`, `partial`, `rejected`) and relay `database_warning`; partial children, detached references, key collisions or overwritten shared entities can coexist with successful generation. Correlate stored rows by `database.entity_keys`, not input spelling. `get_record.database_sync` and `list_database_syncs` provide delivery diagnostics; `list_entity_states` is server-side state only. See [Database sync](database-sync.md).

`sync_records_to_database` revalidates an output against the current published contract. A record ID alone reuses that output; supplying changed output creates a new derived record. Output without a record ID requires `saved_schema_id`. Use this for corrected/rejected results or a run that opted out of sync. It still passes admission and semantic resolution; it cannot force invalid values into a replica.

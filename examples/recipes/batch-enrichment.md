# Recipe: batch-enrich a list of entities

Goal: enrich up to 100 entities in one asynchronous job, with automatic per-entity fusion when
you pick 2+ models, then pull the results back into the chat.

## 1. Pick a schema

> Which Entity Enricher schemas do I have for companies?

`list_schemas` returns compact summaries, pinned schemas first. Grab the `schema_id`.

## 2. Get the entities

Three common sources:

- **Paste them** — "enrich these 20 company names: …". Claude builds the entity list itself.
- **From an external API** — `fetch_entities` performs a server-side GET of a JSON array from
  any REST endpoint (bearer / api_key / basic auth), so a CRM export URL works directly.
- **From past records** — `list_records` + a filter, when re-enriching.

## 3. Start the batch

> Batch-enrich them against the company schema with GPT-5 and Claude Sonnet, English + German.

`start_batch_enrichment` validates plan limits up front and returns `{job_id, total}`
immediately. Per-entity pipelines run in parallel server-side; entities that fail don't block
the rest.

Plan-limit errors are structured — `model_limit_exceeded`, `language_limit_exceeded`,
`concurrent_job_limit_reached`, `insufficient_credits` (HTTP 402) all carry quota details so
Claude can tell you exactly what to reduce (or link you to the billing page).

## 4. Poll

> How is the batch doing?

`get_job_status(job_id)` returns progress counters (completed / failed / total). Jobs live in
a bounded in-memory manager: an **unknown `job_id` means the job finished long ago** — skip
straight to step 5.

## 5. Fetch the results

> Show me the results, failures first.

`list_records(job_id=…)` is the fetch step of every async flow. Then `get_record` for any
record worth a closer look. If some entities failed on specific expertise domains only,
`retry_expertises` re-runs just the failed domains — you don't pay again for what succeeded.

## Cancel

`cancel_job(job_id)` stops a pending/running batch; already-persisted records stay.

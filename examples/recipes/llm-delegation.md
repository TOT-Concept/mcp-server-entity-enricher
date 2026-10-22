<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Delegated LLM calls: run a job on your own model

Start a job with `delegate_llm_calls=true`, answer each of its model calls yourself with `answer_llm_request`, and let the platform validate every answer against the schema.

## What delegation is

Every platform job — an enrichment, a batch, a sample, a schema, a database-model classification, an arbitration — is a sequence of model calls. With `delegate_llm_calls=true` on the start tool, none of those calls reaches a platform model: the job parks each call as a **pending request**, you run it on your own model and hand the result back, and the platform validates it exactly as it validates a provider's answer (output schema, retries, output rules, normalization). The record is stamped with the model key `mcp::client`, costs no credits, and otherwise behaves like any other record: it is validated, saved, and admitted to linked databases under the usual rules.

Delegation is a plan feature (`llm_delegation_not_in_plan` when the plan excludes it). Under delegation every model parameter must be omitted or `auto`: an explicit platform model would run on the platform, so it is refused with `delegation_requires_auto`. `classification_model="auto"` asks you to do the pre-flight classification too; `arbitration_model="auto"` asks you to arbitrate a merge.

Tools that accept the option: `enrich_entity`, `retry_expertises`, `start_batch_enrichment`, `generate_sample`, `create_schema_from_sample`, `classify_database_model`, `create_database_sync` (its classification pass) and `merge_records`. `analyze_sample`, `analyze_schema` and `probe_semantic_concept` run no job and cannot delegate. Benchmarks compare platform models and are not delegated; semantic-ID candidate search uses the organization's embedding model and remains a platform call.

## The loop

1. Call the start tool with `delegate_llm_calls=true`. `enrich_entity`, `create_schema_from_sample` and `merge_records` no longer block: they return the job status as soon as the first call waits for you, like `generate_sample` does. The others return a `job_id` to poll with `get_job_status`.
2. Read `llm_requests` on the status. Each entry is one model call with a `request_id`, a `purpose` naming what the call is for (`<job type>: <step or entity label>`), a `preview` of its last user message (the entity it is about) and an `expires_at` deadline. Several entries can wait at once: a batch runs its entities in parallel, a schema generation runs several analysis branches. Only the **first** entry is complete (`complete: true`): the MCP `sampling/createMessage` shape — `systemPrompt`, `messages`, `tools`, `toolChoice`, `maxTokens`, optionally `temperature` — plus its `attachments` list. The others are envelopes (`message_count`, `tool_names`, `attachment_count`), because each repeats the same system prompt and schema and five of them do not fit a client's response budget. Answer the first and the next one comes back complete, or read any of them complete with `get_job_status(job_id, llm_request_id=...)`.
3. Run the request on your own model, as faithfully as a provider would: the system prompt and the whole conversation, with the listed tools. When `toolChoice.mode` is `required`, your answer is exactly one `tool_use` block for the listed tool whose `input` conforms to its `inputSchema` — that schema is the platform's own output contract for the call. When it is `auto` (the expert-domains strategy), you may call the `submit_*` tools and finish with a text answer.
4. Call `answer_llm_request(job_id=..., request_id=..., response=...)` with the MCP `CreateMessageResult` of your model: `{"role": "assistant", "model": "<your model>", "stopReason": "toolUse" | "endTurn", "content": [{"type": "tool_use", "id": "...", "name": "...", "input": {...}}]}` (or a single `{"type": "text", "text": "..."}` block). Add `usage: {"input_tokens", "output_tokens"}` when you know them. The call waits up to `wait_seconds` and returns the next status: more pending requests, a question pause, or the terminal result with its records.
5. Repeat until the status is terminal. `cancel_job` abandons the job and fails every pending request at once.

Never invent a value the request does not ask for, and never answer a request with the result of another one: the `request_id` binds the answer to its conversation.

## When an answer is rejected

The platform validates your `tool_use` input against the schema and its rules. A rejected answer is not an error of the tool call: `answer_llm_request` succeeds, and the **same conversation comes back as a new pending request** whose last user message is a `tool_result` with `isError: true` carrying the validation error (the status also summarizes it as `rejected_answer_error`). Read the error, fix the input, answer again. The number of re-asks is the job's own retry budget; past it the call fails as it would for any model.

Enrichment behaves slightly differently on a rejected output: instead of re-asking the whole output, the platform asks a **new, smaller request** limited to the rejected properties (a repair call, named so in `purpose`). Answer it like any other request.

Record as a product finding, not as your own mistake: a rejection whose error does not match the `inputSchema` you were given, a request too large or truncated to be executed, a request that expires while you are answering it, or a `purpose` that does not say what is expected.

## Attachments

Text-extracted documents (DOCX, XLSX, PPTX, HTML, CSV, TXT, Markdown…) are already inside `messages`, exactly as a platform model receives them. Binary documents (PDF, images, audio) are never inlined: the request lists them under `attachments` with `attachment_id`, `filename`, `media_type`, `size_bytes`, `sha256`, `delivery: "resource"` and a `uri` — `enricher://attachments/{attachment_id}`, an MCP resource serving the bytes with their real media type. Read it once with `resources/read` and reuse it: the list is **identical for every request of the job**, and the bytes behind an id never change. The messages name each binary with a placeholder pointing at its `attachments[i]` entry.

Documents you hold on your own side (a page from your own connectors, a file in your context) need nothing from the platform: answer from what you have. The platform cannot list or verify such sources, so say so when you report evidence.

## Timeouts and the job status

A pending request expires 15 minutes after it was parked (`expires_at`). An unanswered request fails the job with `error_code: llm_request_timeout` and fails its other pending requests with it — a client that stopped answering has abandoned the job. There is no default answer for a model call, so this timeout never "continues with defaults" like a question pause does.

While requests are pending the job is `paused`, and `pending_llm_requests` on the status counts them. `answer_job_question` is for question pauses only; on a job paused on LLM requests it answers `llm_request_pending` and points you back here. `get_job_status(events_after=0)` shows the `llm_request`, `llm_request_answered` and `llm_request_timeout` events in order.

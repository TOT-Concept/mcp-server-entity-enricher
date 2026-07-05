# Recipe: author a schema from a sample, then enrich

Goal: go from "I want structured data about *X*" to a saved, reusable enrichment schema and a
first enriched record — entirely inside one chat.

Requires the **editor** role (or higher) — your own role when connected via OAuth, or the
key's role when using an API key.

## 1. Generate a sample entity

> Generate a sample entity for "biotech company" with Entity Enricher — include identity,
> pipeline, financials and key people.

Claude calls `generate_sample`, which is **asynchronous**: it returns a `job_id` and waits
server-side for the next decision point. Two outcomes:

- **Completed** — the sample JSON is in the response.
- **Paused** — if you attached source documents, the interactive planner may ask structural
  questions ("one product per sample, or the whole catalog?"). Claude relays them to you and
  resumes with `answer_job_question(job_id, answers)`. This can repeat for several rounds.

## 2. Turn the sample into a schema

> Looks good — create a schema from that sample.

Claude calls `create_schema_from_sample` (synchronous, LLM-backed, auto-saved). The result
includes the new `schema_id` and the generated JSON schema with expertise domains, key
properties, and nullable flags inferred from the sample data.

## 3. Refine it

> Rename the schema to "Biotech Company v1", and make the clinical-pipeline section
> multilingual in English and French.

- Structural edits in natural language → `edit_schema` (LLM-backed, in place).
- Rename / retag / pin → `update_schema` (no LLM call, free).
- If Claude authored the schema JSON itself in the chat, `save_schema` persists it with
  server-side validation and **zero LLM cost**.

## 4. Enrich against it

> Enrich "Moderna" against the Biotech Company v1 schema with two models and fuse the results.

Claude calls `enrich_entity`. Watch for the classification gotcha: if you enabled a
classification model and the entity doesn't match the schema type, the tool returns a
**non-error** response with `error_code: "classification_warning"` and the classifier's
reasoning. Confirm, and Claude retries with `force_after_classification_warning=true`.

## 5. Inspect

> Show me the record.

`get_record` returns the full structured output, validation errors (if any), token usage and
cost. Or Claude may read the `enricher://records/{record_id}` resource directly.

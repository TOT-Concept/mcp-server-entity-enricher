<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Documents and source-grounded samples

Choose extraction, knowledge enrichment or a two-pass combination, and preserve attachment provenance across calls.

## Upload once and reuse

`upload_attachment` takes a filename and base64 bytes without a `data:` prefix. It returns `id` and `requires_capability`; pass IDs through `attachment_ids`. The server detects content and applies the administrator's MIME policy: extract to text or send model-readable binary. Formats include PDF, PNG/JPEG, MP3/WAV/M4A/OGG/FLAC, DOCX/ODT/RTF/DOC, XLSX, PPTX, EPUB, HTML, CSV, TXT and Markdown. Actual acceptance depends on policy and file validation.

Leave model selection on `auto` unless an explicit model is needed: it intersects task and attachment requirements. A listed capability is nominal, not proof that the provider accepts every combined media/tool/output mode or has quota available. Uploading a file does not run a model; consuming it in a generation/enrichment can incur charges.

Keep attachments required by saved schemas, future enrichments or regeneration. `delete_attachment` removes the stored source permanently. Existing records do not disappear, but the source is no longer reusable. Cleanup is a deliberate retention decision, not a mandatory step after each run.

## Sample modes

Without `attachment_ids`, `generate_sample` designs an example using model knowledge. `enable_web_search=true` can ground external facts if supported. With attachments, it extracts the document or describes observable image attributes; external search does not relax the source-only rule. This restriction describes **sample generation**: attachment-backed `enrich_entity` follows its enrichment prompt and schema, not a general promise of source-only extraction.

Multiple attachments are not an instruction to return one record per file. The sample planner distinguishes complementary pages/views of one entity from different instances of a common type. In the first case it combines values; in the second it chooses one reference file under their common type. It can ask to exclude an outlier; incoherent sources fail with `incoherent_attachments`. Inspect the terminal `attachment_coherence` verdict. Attachments force `sample_count` to one, even when several files were uploaded.

In either mode a materially ambiguous request can pause. The response's `pause` contains questions. `answer_job_question.answers` maps each question ID to `{"option_ids": [...], "text": ...}`; omitted questions use defaults. Relay decisions to the user unless the choices/defaults were already authorized. `auto_answer=true` makes sample generation autonomous, accepting those interpretations. After a wait window, a running response is normal: continue through `get_job_status`.

## Extraction plus external research

For a request combining media identification and research, first generate a source-mode sample with search off. Confirm the identity or uncertainty it reveals. Then call `generate_sample` without attachments, using the confirmed identity in the request or `typical_objects`, with web search enabled. The two calls produce separate records. Combine them explicitly in the conversation if needed; retain which facts came from the source and which were researched. Do not portray a guessed identity as confirmed.

This is not a switch enabling web search in enrichment: the MCP `enrich_entity` and `start_batch_enrichment` tools currently do not expose that option. Do not claim those calls researched the web. The REST/UI surface has options the MCP wrappers do not expose.

## Carry sources into a schema

After reviewing a successful sample, pass its `sample_record_id` to `create_schema_from_sample` to inherit all stored samples and linked attachments. Add edited `entity_samples` to replace the JSON while preserving attachment inheritance. Explicit `attachment_ids=[]` means no inherited attachments; a nonempty list replaces them. Generation returns links to the saved schema and record. Continue with [Schema from samples](schema-from-sample.md).

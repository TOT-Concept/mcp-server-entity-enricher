<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Schema from samples

Design a reusable schema from reviewed examples, including identity, relationships and multilingual fields.

## Choose the entry point

If you already have representative JSON instances, pass them to `create_schema_from_sample` as `entity_samples`. If you already have a schema, use `save_schema`; a sample instance and a schema document are different inputs. Otherwise use `generate_sample` to design examples from a request or uploaded sources. Schema authoring requires the editor role. Generation and analysis incur model costs; mechanical edits do not call a model.

## Generate representative samples

Write the entity type, required information, scope and size/depth budget in `generate_sample.request`. Pass the number of instances separately in `sample_count` (1–20). Each sample is one instance of the same type, not a wrapper around several instances. Three varied samples are a useful starting point for schema generation: repeated related objects under different parents can expose fields that belong to the relationship. This is a recommendation, not a requirement or proof that all relationships are correctly modeled.

`typical_objects` names concrete instances; remaining slots are chosen together by the first generation. The first sample establishes the field set; subsequent variants reuse it. Inspect `samples_note` for under-delivery or an attachment-imposed cap. With attachments, generation is source-grounded and capped at one sample; read [Documents](documents.md).

The job can return running, paused or completed. Relay questions under `pause` through `answer_job_question`; otherwise poll `get_job_status`. Knowledge mode can pause too. `auto_answer=true` authorizes the generator to use default interpretations without pausing. A failed job is not an approved sample. Retrieve persisted sample-generation records through the returned record IDs or `list_records(job_id=...)`.

## Review the contract

Review scope, property names, types, representative missing fields, arrays, identities and relationships. Group consequential changes into one proposal. Do not silently change factual values or structure; a choice already explicitly authorized need not be asked again.

A scalar naming a reusable real-world object remains a value. A keyed nested entity can have its own table and identity. Keep facts about that entity separate from facts about its relationship to the parent: a role or a per-parent designation belongs to the relationship. Knowledge-mode generation may restructure these sites before review and reports changes in `warnings`; source mode respects the source shape.

Samples contain one language, not objects keyed by language code. Set a property's `multilingual` flag through `update_schema_property` after generation when needed. Enrichment's `languages` controls the requested translations. `create_schema_from_sample` has no free-text `instructions` parameter.

Optional `analyze_sample` reports ambiguous or unmappable names, missing unit/period/range context, and mixed entity/relationship facts. It persists an analysis record but leaves the sample unchanged. Review suggested names and structural corrections before using them; this pass is not a mandatory gate.

## Generate and inspect

Call `create_schema_from_sample` with one or more `entity_samples`, a successful `sample_record_id`, or both. Explicit samples replace the stored samples; omitted `attachment_ids` inherit the record's sources, while `[]` deliberately removes them. The schema covers the union of observed fields; missing or null observations make fields nullable. Use consistent field names, including across items of an array; completely disjoint item fields are rejected.

Decide `generate_semantic_ids` before generation when repeated entities need resolution across runs. It requires an organization embedding model and adds embedding cost; obtain agreement unless already authorized. Stable machine identifiers can also make good database keys. Human labels may vary and create duplicate rows; semantic resolution can itself make wrong matches, so inspect its warnings. Retrofitting semantic IDs requires regenerating or editing the relevant objects.

Generation preserves the reviewed relationship structure and reports residual identity-scoping problems as annotations. It can canonicalize names/values: fold Latin diacritics from property names, move embedded units into names, normalize unsupported dates to integer years, and collapse per-language sample objects. Inspect the saved sample and returned warnings rather than assuming byte-for-byte identity.

Compare the schema to the reviewed samples. Use `get_schema_part` and property tools for approved targeted edits, `update_schema` for a full replacement, or edit samples and regenerate when the sample contract itself must change. There is no MCP tool named `edit_schema`. Generated vocabularies are open by default; close only an exhaustive set. See [Schema reference](schema-reference.md), then [Enrichment and fusion](enrichment-and-fusion.md).

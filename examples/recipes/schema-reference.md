<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Schema format and targeted editing

Read, author and edit Entity Enricher schema documents without confusing serialized JSON Schema, sample data and property-tool paths.

## Document format

`schema_content` and `target_schema` accept the supported `GeneratedJsonSchema` model. Its serialized form declares JSON Schema 2020-12 and places `title`, `type: "object"` and `properties` at the top level. Entity Enricher adds behavioral annotations and extension sections; arbitrary JSON Schema constructs are not necessarily understood by its enrichment and relational projection engines.

This minimal document can be passed as `save_schema.schema_content` (with a separate saved-schema `name`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Catalog item",
  "type": "object",
  "description": "An item identified by its catalog code.",
  "properties": {
    "code": {"type": "string", "identifying": true},
    "description": {"type": ["string", "null"]}
  }
}
```

| Serialized section or reference | Meaning |
|---|---|
| `$defs`, `#/$defs/Type` | Reusable entity definitions; references represent relationships. |
| `x-enums`, `#/x-enums/Name` | Named scalar vocabularies, with `enum`, optional `value_descriptions`, `closed` and `rejected_values`. |
| `x-localized`, `#/x-localized/LocalizedString` or `LocalizedStringArray` | Derived language-map definitions for multilingual text. |
| `x-expertiseDomains` | Named expertise domains used to route properties to model calls. |
| `x-entityMap` | Entity/relationship regions and pending unification proposals. |

Internally the model has a `root` envelope, entity `name`, `$enums` and `expertise_domains`. The parser also accepts that internal representation, but API/MCP schema documents serialize to the format above. Follow the returned document when replacing it. An enum's `name` remains part of its definition; entity names serialize as `title`.

Nullable scalar properties serialize as a type array including `null`; nullable references use `anyOf` with a null branch. Multilingual text becomes a reference to a generated language-map definition. Do not manually construct per-language sample objects to request translations.

**Unknown keywords are dropped, not rejected.** A conformant reader ignores keywords it does not know, so a full-document write (`save_schema`, `update_schema`) never fails on one — it reports them instead: `ignored_keywords` lists every dropped keyword with the node `path` (`''` = document root, `$defs.Type`, `players[].name`) and a `hint` naming the level it is read at, and `applied_repairs` carries the summary line. Read it after every write. The typical miss is a **property flag placed on an object**: `semantic_id`, `semantic_concept_type` and `semantic_source_keys` on a `$defs` entity configure nothing — they belong on the property inside that entity that carries the id. Standard keywords this dialect does not read are reported too (`required` is derived from nullability, an inline `enum` must be an `x-enums` vocabulary, combinators other than the nullable `anyOf` are dropped).

## Behavioral flags

| Flag used by property tools | Effect |
|---|---|
| `identifying` | Describes entity identity; guides enrichment input and key selection. |
| `preserve` | Value belongs to the caller and must be supplied; enrichment does not research it. |
| `nullable` | Allows missing knowledge; omitted/false requires a value at enrichment. |
| `multilingual` | Localizes text or text arrays using enrichment's requested languages. |
| `expertise` | Routes a property to a declared expertise domain. |
| `format`, `pattern` | Value format and validation constraints. |
| `semantic_id`, `semantic_source_keys`, `semantic_concept_type`, `semantic_embedding_model`, `judge_floor` | Semantic identity and resolution configuration. |
| `database_key`, `db_type`, `db_type_length`, `index`, `unique_group` | Relational identity, SQL typing and indexing. |
| `shared`, `ordered` | Relationship ownership and array order semantics. |

Not every flag is freely mutable through the property tool; its `flags` description lists accepted updates. Identity-source changes can require a full schema edit. Server validation may reject or normalize illegal combinations; inspect `applied_repairs`, and `ignored_keywords` on full-document writes. A missing `preserve` field must not be invented. If a field is supposed to be researched, reconsider whether `preserve` belongs on it.

## Read the correct version

`get_schema` defaults to the working copy. For a database-linked schema, request `version="published"` to inspect the document enrichment uses; the computed `input_contract` describes the enrichment contract even when reading the working copy. A linked schema without its first publication is not ready to sync. Unlinked drafts use their working copy directly.

`input_contract.identifying_keys` guides naming; other input names are accepted. All `preserve` paths and keys for each supplied array item are enforced. An input carrying no value is refused. `input_contract_unsatisfied` returns the offending paths.

## Targeted edits

Use `get_schema_part` to avoid a full document round trip. No path gives the index; `$defs.Type` or `$enums.Name` reads a named definition. These are **property-tool paths**, so `$enums.Name` remains the path spelling even though the serialized section is `x-enums`. Object/leaf paths are dot-separated and `[]` enters an array item, for example `contributors[].person.name`. The empty parent path means the root.

Read the relevant card, then use `update_schema_property`, `add_schema_property` or `move_schema_property`. Editing a shared `$defs` type affects all its usage sites. A null entry in `flags` clears that flag. Moves and removals refuse protected identity members or recursive containment; recompose identity before removing a source key.

Renaming carries property order and identity-source references. For published schemas it records `renamed_from` migration intent; `db_name` is not blindly rewritten. Inspect the publish preview for the actual SQL consequences. Structural edits require `publish_schema` when linked; neutral changes propagate automatically. See [Database sync](database-sync.md).

`analyze_schema` writes ambiguity and identity-scoping annotations; it does not perform suggested structural edits. Use a precise description to resolve ambiguous meaning on an established contract. A rename is possible through the property tools, but can require a migration. `resolve_unify_proposal` previews accepting or dismissing a cross-site entity-type proposal: acceptance combines sites under a winning definition and may make fields nullable. Review the returned document before persisting.

`get_enum_candidates` reports out-of-set observations for open vocabularies. Edit `x-enums` through a full `update_schema` replacement to admit members, record `rejected_values`, or set `closed=true` only when the vocabulary is exhaustive. A rejected candidate is hidden from the review list, not prohibited as model output.

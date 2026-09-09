<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Semantic identities and vocabulary curation

Recognize recurring entities across surface forms, review uncertain matches and understand how vocabulary changes affect replicas.

## Resolution is more than similarity

A semantic ID belongs to an organization's concept vocabulary. Resolution combines known aliases, embedding neighbors and, where available, an identity judge. Similarity selects candidates; it does not prove that two texts refer to the same entity. `judge_floor` is the similarity floor for sending a candidate to the judge, not an unconditional merge threshold.

Each vocabulary slice is defined by concept type and embedding model. Similarity values are comparable only within that slice. Read `list_semantic_concepts` for concepts, aliases, usage counts and type/model facets; `get_semantic_concept` supplies authoritative same-slice neighbors and alias IDs.

`view="review"` lists pairs the judge left for a person: uncertainty, possible duplicate candidates or an unusually close pair judged distinct. It is not a list of every similar pair. Review the identity evidence before merging; a high similarity score by itself is insufficient.

## Add, alias or import

Use `probe_semantic_concept` before adding. Its outcomes are `exact_hit`, `match` or `no_match`; inspect the incumbent and judge evidence when present. If a matching concept already covers the intended object, reuse it. Do not keep retrying an add refused with `concept_exists`.

`add_semantic_concept` creates a concept at zero usage, or adds a surface form when `alias_of` is supplied. An alias resolving to a different concept is refused. `update_concept_alias` removes an alias or promotes it to canonical; the last alias cannot be removed through that tool. Removing an alias stops that spelling resolving through the alias, but future semantic resolution may still match it.

Probing/adding requires editor. Exact known texts may avoid embedding work; other resolution can incur embedding and identity-judge cost. Preview does not mean free. `embedding_model` seeds a new concept type's space; it cannot switch an existing type to another model.

For 1–1000 texts, use `import_semantic_concepts(mint=false)` first. The report shows exact, matched and would_mint outcomes without minting misses. Review would_mint rows before `mint=true` (owner required; reporting is editor). A new type may be created through this flow. The resolution process still runs in report mode.

## Merge or delete

`merge_semantic_concepts` defaults to `impact_only=true`. Review affected aliases, entities, references and replicas, then obtain approval for execution with `impact_only=false` (owner). The loser’s aliases resolve to the winner, entity identities/references are rewritten, and convergence deltas reach linked replicas asynchronously.

`delete_semantic_concepts` also defaults to impact reporting. Execution requires editor; clearing whole types requires owner. Deleting concepts can break convergence with IDs already stored in replicas: future enrichments may mint new IDs. Unused-only cleanup still removes vocabulary, even though current record usage is zero. Review impact and authorization before deleting. A type disappears when its last concept is removed.

Enrichment `identity_merges` and `identity_underidentifies` warnings are review triggers, not commands to merge. Compare the texts and identity-source keys. A poorly composed identity may need schema correction and re-enrichment instead of vocabulary surgery. Shared-entity overwrites may instead reveal relationship-specific facts modeled as global ones; see [Database sync](database-sync.md).

## Change embedding models

Use `migrate_semantic_embeddings` for existing vocabularies. `status` reads the transition. `preview` (owner) estimates cost and identifies potential collisions under `target_model`; inspect them before authorizing `start`. `source_model` and `concept_types` optionally scope the migration.

Starting stages new vectors while enrichment continues, then switches the selected slices after coverage is complete. It is resumable after interruption. Cancellation marks the transition cancelled; it does not itself forcibly interrupt an in-flight embedding task or roll back a completed cutover. Inspect status and slice assignments before assuming work stopped. Migration and preview can require embedding work. Organization model configuration and other administration remain in the web app where no MCP tool exposes them.

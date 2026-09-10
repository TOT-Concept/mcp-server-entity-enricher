<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Database sync and migrations

Turn enrichment schemas into relational tables in your own database and verify admission, migration and replica delivery separately.

## Model identity and ownership

Register a saved schema with `create_database_sync` (owner plus a plan with database sync). Dialects are `postgres`, `mysql` and `sqlite`. Entity Enricher supplies schema/data changes; `ee-database` or another consumer applies them. Your target DSN stays on the replica host.

Before registration, review entity identity and relationships. Semantic IDs help repeated descriptions converge, but require an organization embedding model and add cost. Stable machine IDs can also serve as keys. Names/websites may drift. The deterministic key ladder prefers semantic identity, Id-like fields, natural identity and owned-content/reference identities. Inspect returned `stamped_keys` rather than assuming generation chose sufficient keys.

An owned object's identity includes its parent; a shared object converges across parents. Relationship-specific facts must not be stored as globally shared entity facts. Two plausible children of one parent must not share keys while differing by an edition, location or other discriminator. An owned relationship item pointing to a shared entity is a junction with attributes: its identity is (owner, reference), reported as `owner_keys` + `reference_keys` in `stamped_keys` — `database_keys: []` there means the reference is the identity, not that the row has none. A `database_key` on such an item is a discriminator that extends that identity (the same referenced entity twice under one owner, in different roles), never a replacement for the reference.

Owned child arrays replace previous membership: a child omitted by a later answer is deleted from the replica, even if the shorter answer merely forgot it. This is not a union across runs. Keep an independent history if that is required. Unchanged rows do not need rewriting, and a no-change enrichment may queue no delta. `_sync_revision` tracks changed rows, but a row-only incremental query cannot recover deleted rows; use the delta feed when deletions matter.

## Register and review

Registration links the schema **unpublished** and normally starts a billed database-model classification job. Poll `classification_job_id`, or inspect `classification_skipped`. A skipped or failed pass leaves relationship sites without a `shared` verdict, and `publish_schema` refuses until every site has one: re-run `classify_database_model`, or set `shared` with `update_schema_property`. Then read `get_schema`'s working copy: review keys, ownership, SQL types, search intent and entity-level indexes. `classify_database_model` is incremental after relevant edits, not a forced full rerun of unchanged fields. Correct proposals with property tools.

Review `registration_notices` and `custody_warning` before publication. Defaults commit to choices even when omitted:

| Option | Consequence |
|---|---|
| `pk_strategy="surrogate"` | Physical surrogate IDs with unique natural keys; the strategy locks once the physical model ships. Natural-key mode restricts later re-keying. |
| `on_gaps="skip_children"` | Drops incomplete child items or detaches incomplete shared 1–1 references; uncontainable gaps still reject the entity. |
| `on_gaps="reject_entity"` | Any required gap rejects the whole entity. |
| `on_gaps="accept_partial"` | Writes gaps as null; later partial values can erase earlier values. |
| `propagate_not_null` | Automatic under strict gap policies; true is incompatible with accept_partial. Changing constraints after delivery can need a validating migration. |
| `purge_on_ack` / `purge_entity_state` | Deletes delivered copies immediately or after configured delays. Entity-state purge transfers custody to replicas and makes snapshots incomplete. |

The database owns the multilingual key-language lock. Registration usually chooses it; publication can request it when classification first reveals a multilingual key. Every linked schema must agree. Purged entity state also limits server-side duplicate checks; a later constraint conflict may be caught and quarantined by the replica instead.

## Publish and connect

Use `publish_schema(validate_only=true)` to inspect blockers, warnings and exact per-database `migration_sql`. The first actual publication starts the schema/data feed. Structural edits use the same preview/publication process. Transform migrations require approval of the concrete changes before `confirm_transforms=true`; already authorized changes need not be reconfirmed. Cross-schema disagreement remains a blocker. Neutral edits propagate automatically.

Registration may assign one connected managed host automatically; `target_host` chooses explicitly. With several candidates use `assign_sync_host` after choosing. A managed host can provision the physical database and start syncing. Moving hosts revokes the old host credential, but does not evict a manual pairing.

Without managed provisioning, use `create_database_credential` and its returned install/pair/run commands on the intended replica host. Reissuing revokes an existing credential and disconnects that client. The returned token is shown once. The client bootstraps from a snapshot, then consumes ordered changes; use its returned command suggestions rather than inventing paths or secrets.

## Read the real outcome

An enrichment can generate successfully yet be `database.status="partial"` or `"rejected"`. Inspect and relay missing fields, skipped children, detached references, key collisions, shared-entity overwrites and identity warnings. `database.entity_keys` identifies the stored rows; input text may have been canonicalized. Fix the schema/output before re-enrichment or `sync_records_to_database`; dropped data is not magically resent.

`list_entity_states` shows current **server-side** rows, not delivery confirmation. `get_record.database_sync` includes per-replica delivery state. `list_database_syncs` reports pending and quarantined deltas plus projection-upgrade blockers. Pending zero alone is insufficient. A consumer acknowledgement reports application; inspect the replica itself when independent confirmation is required.

## Consume deltas directly

`fetch_database_deltas(claim=false)` is a replayable read. `claim=true` leases a window for 120 seconds; the registration's page limit also applies. Apply the whole returned window in one transaction, including DDL, in order. Windows preserve enrichment batch boundaries. Advance your cursor and call `ack_database_deltas(up_to_id=...)` only after successful application. Acknowledging can permanently purge data: never acknowledge failed or skipped statements. Expired leases can be redelivered; use a consumer with appropriate transaction/replay handling.

When `snapshot_required=true`, reapply the registration's snapshot through the replica client/REST flow before resuming. The MCP does not expose a snapshot-download tool. Other operational settings, linking additional schemas, quarantine administration and projection-upgrade confirmation may require the web app or REST API.

`delete_database_sync` stops the feed and removes queued deltas, leaving external tables unchanged. Entity state and schema flags remain unless explicit teardown options are approved. [Enrichment and fusion](enrichment-and-fusion.md) describes recovery and corrected-output injection.

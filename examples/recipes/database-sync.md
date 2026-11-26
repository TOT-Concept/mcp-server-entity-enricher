<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Database sync and migrations

Turn enrichment schemas into relational tables in your own database and verify admission, migration and replica delivery separately.

## Model identity and ownership

Register a saved schema with `create_database_sync` (owner plus a plan with database sync). The target is PostgreSQL (`dialect="postgres"`, the default) or SQL Server 2022 or later (`dialect="sqlserver"`); the snapshot and every delta are written in its SQL. Entity Enricher supplies schema/data changes; `ee-database` or another consumer applies them. Your target DSN stays on the replica host.

The database type is chosen at registration. An owner can change it later in the web app ("Change target database"): the sync then restarts on a new, empty database, queued deltas are discarded, a manually paired client must be paired again (a managed host that serves the new type claims the sync by itself), and `snapshot_required` stays true until the snapshot is downloaded (by the sync client or through REST). Downloading clears it whether or not the script was applied, so apply the snapshot you download before fetching deltas. A consumer that reads the feed itself (MCP, REST, n8n, Make) is not disconnected: left as it is, it applies the new feed to the previous database, so point it at the new database before downloading the snapshot. No MCP tool changes the target.

Before registration, review entity identity and relationships. Semantic IDs help repeated descriptions converge, but require an organization embedding model and add cost. Stable machine IDs can also serve as keys. Names/websites may drift. The deterministic key ladder prefers semantic identity, Id-like fields, natural identity and owned-content/reference identities. Inspect returned `stamped_keys` rather than assuming generation chose sufficient keys.

An owned object's identity includes its parent; a shared object converges across parents. Relationship-specific facts must not be stored as globally shared entity facts. Two plausible children of one parent must not share keys while differing by an edition, location or other discriminator. An owned relationship item pointing to a shared entity is a junction with attributes: its identity is (owner, reference), reported as `owner_keys` + `reference_keys` in `stamped_keys` — `database_keys: []` there means the reference is the identity, not that the row has none. A `database_key` on such an item is a discriminator that extends that identity (the same referenced entity twice under one owner, in different roles), never a replacement for the reference.

Owned child arrays replace previous membership: a child omitted by a later answer is deleted from the replica, even if the shorter answer merely forgot it. This is not a union across runs. Keep an independent history if that is required. Unchanged rows do not need rewriting, and a no-change enrichment may queue no delta. `_sync_revision` tracks changed rows, but a row-only incremental query cannot recover deleted rows; use the delta feed when deletions matter.

## Register and review

Registration links the schema **unpublished** and normally starts a billed database-model classification job. Poll `classification_job_id`, or inspect `classification_skipped`. A skipped or failed pass leaves relationship sites without a `shared` verdict, and `publish_schema` refuses until every site has one: re-run `classify_database_model`, or set `shared` with `update_schema_property`. Then read `get_schema`'s working copy: review keys, ownership, SQL types, search intent and entity-level indexes. `classify_database_model` is incremental after relevant edits, not a forced full rerun of unchanged fields. Correct proposals with property tools.

The MCP registration response never returns the database webhook signing secret. Configure, reveal or rotate webhook credentials in the web app instead.

Review `registration_notices` and `custody_warning` before publication. A notice with `option="dialect"` counts the indexes or foreign keys the schema declares that SQL Server does not create (substring-search, point and range indexes, an index on text without a length, a foreign key inside a cycle of references): the data is delivered all the same, relay what is absent. Defaults commit to choices even when omitted:

| Option | Consequence |
|---|---|
| `pk_strategy="surrogate"` | Physical surrogate IDs with unique natural keys; the strategy locks once the physical model ships. In natural-key mode a later re-key also rewrites the copies of the key in every dependent table. |
| `on_gaps="skip_children"` | Drops incomplete child items or detaches incomplete shared 1–1 references; uncontainable gaps still reject the entity. |
| `on_gaps="reject_entity"` | Any required gap rejects the whole entity. |
| `on_gaps="accept_partial"` | Writes gaps as null; later partial values can erase earlier values. |
| `propagate_not_null` | Automatic under strict gap policies; true is incompatible with accept_partial. Changing constraints after delivery can need a validating migration. |
| `purge_on_ack` / `purge_entity_state` | Deletes delivered copies immediately or after configured delays. Entity-state purge transfers custody to replicas and makes snapshots incomplete. |

The database owns the multilingual key-language lock. Registration usually chooses it; publication can request it when classification first reveals a multilingual key. Every linked schema must agree. Purged entity state also limits server-side duplicate checks; a later constraint conflict may be caught and quarantined by the replica instead.

## Publish and connect

Use `publish_schema(validate_only=true)` to inspect blockers, warnings and exact per-database `migration_sql`. The first actual publication starts the schema/data feed. Structural edits use the same preview/publication process. Transform migrations require approval of the concrete changes before `confirm_transforms=true`; already authorized changes need not be reconfirmed. A SQL Server database may also list `migration_sql[].physical_changes` (a text column that becomes a key or gets an index is narrowed to 400 UTF-16 units): they need the same approval, and `requires_confirm` is true for them even when `diff.transforms` is empty. `unserved`, in the preview and in the result, lists per SQL Server database the indexes and foreign keys the schemas declare that it does not create — `notices` says each kind's count and cost: relay them, since classification can ask for a substring search after `registration_notices` were issued. Cross-schema disagreement remains a blocker. Neutral edits propagate automatically.

Registration may assign one connected managed host automatically; `target_host` chooses explicitly. With several candidates use `assign_sync_host` after choosing. A managed host can provision the physical database and start syncing. Moving hosts revokes the old host credential, but does not evict a manual pairing.

Without managed provisioning, use `get_database_setup_instructions` and run its install/pair/run commands on the intended replica host. Pairing opens the browser confirmation page: the owner chooses the database, and the credential travels directly to the polling CLI rather than through the MCP client. Confirming a new pairing replaces that database's previous credential and disconnects its prior client. The client bootstraps from a snapshot, then consumes ordered changes; use the returned command suggestions rather than inventing paths or secrets.

## Read the real outcome

An enrichment can generate successfully yet be `database.status="partial"` or `"rejected"`. Inspect and relay missing fields, skipped children, detached references, key collisions, shared-entity overwrites and identity warnings. A SQL Server database also refuses an entity whose key or indexed text (items of a text list included) is longer than 400 UTF-16 units, whose key (or item of an unordered text list) ends with a space, or which holds a number outside `DECIMAL(38,10)`; the reason names the table and column. `database.entity_keys` identifies the stored rows; input text may have been canonicalized. Fix the schema/output before re-enrichment or `sync_records_to_database`; dropped data is not magically resent.

`list_entity_states` shows current **server-side** rows, not delivery confirmation. `get_record.database_sync` includes per-replica delivery state. `list_database_syncs` reports pending and quarantined deltas plus projection-upgrade blockers. Pending zero alone is insufficient. A consumer acknowledgement reports application; inspect the replica itself when independent confirmation is required.

## Consume deltas directly

`fetch_database_deltas(claim=false)` is a replayable read. `claim=true` leases a window for 120 seconds; the registration's page limit also applies. Each delta's `sql` is one executable batch in the registration's dialect: run it as a single statement batch. Apply the whole returned window in one transaction, including DDL, in order, on one connection. On SQL Server never join two deltas into one batch: a batch is compiled whole, so a data delta that names a column the previous schema delta adds does not compile. A client that can send only one query wraps each delta in `EXEC(N'…')` (every `'` doubled) inside `SET XACT_ABORT ON; BEGIN TRY BEGIN TRANSACTION; … COMMIT; END TRY BEGIN CATCH IF @@TRANCOUNT > 0 ROLLBACK; THROW; END CATCH`. Windows preserve enrichment batch boundaries. Advance your cursor and call `ack_database_deltas(up_to_id=...)` only after successful application. Acknowledging can permanently purge data: never acknowledge failed or skipped statements. Expired leases can be redelivered; use a consumer with appropriate transaction/replay handling.

Before its first window a new replica needs the snapshot: it creates the tables, which the feed does not. When `snapshot_required=true`, reapply the registration's snapshot through the replica client/REST flow before resuming. The MCP does not expose a snapshot-download tool.

A SQL Server snapshot, and the REST feed in `format=sql`, are `sqlcmd` scripts: batches separated by lines holding only `GO`, which is not T-SQL. `sqlcmd -S host,1433 -d mydb -U user -I -b -x -i snapshot.sql` runs one as is: `-d` names the target database (without it `sqlcmd` writes to the login's default database, usually `master`), `-I` turns quoted identifiers on, `-b` stops at the first error, `-x` leaves `$(…)` in the data alone, and `-C` is needed when the server's certificate is self-signed. To apply one yourself, split it on those lines, run the batches in order on one connection and stop at the first error; the snapshot's first batch sets `XACT_ABORT`, `QUOTED_IDENTIFIER` and `ANSI_NULLS` for the session (do not reset it between batches), and each delta of the feed sets the first two itself. The snapshot opens and commits its own transaction; the `format=sql` feed does not, so add `BEGIN TRANSACTION;` and `GO` before it, `GO` and `COMMIT;` after it. A PostgreSQL snapshot is one script: `psql "postgres://user:pass@host:5432/mydb" -v ON_ERROR_STOP=1 -f snapshot.sql`. Other operational settings, linking additional schemas, quarantine administration and projection-upgrade confirmation may require the web app or REST API.

When you read a SQL Server replica:

- Text compares exactly, case included (collation `Latin1_General_100_BIN2`), and a join with a column of another collation needs `COLLATE DATABASE_DEFAULT` on one side.
- A required column outside the keys is enforced by the check `nn_<table>_<column>` and reads as nullable in the catalog.
- Key and indexed text holds 400 UTF-16 units, as a declared length counts them: an emoji counts as two.
- A numeric column keeps 10 decimals; a `double` one is not limited to them.
- An indexed multilingual field has one column per language, `<field>_<language>` (its first 400 characters): filter on that column, since a filter on `JSON_VALUE` does not use the index. In a JSON path, quote the language: `JSON_VALUE([name], '$."zh-CN"')`.
- Index names starting with `idx_` or `uq_` are reserved for the sync: a migration may drop an index with such a name, so give yours another prefix.

`delete_database_sync` stops the feed and removes queued deltas, leaving external tables unchanged. Entity state and schema flags remain unless explicit teardown options are approved. [Enrichment and fusion](enrichment-and-fusion.md) describes recovery and corrected-output injection.

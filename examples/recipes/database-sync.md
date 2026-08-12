# Recipe: from a schema to your own database, and keep it converged

Goal: turn a saved schema into real relational tables in **your** PostgreSQL (or MySQL, or
SQLite), have every enrichment land there, and let later schema changes arrive as migrations —
driven from one chat.

Requires the **editor** role (owner to register the database on some plans) and a plan that
allows at least one database sync.

## 0. Get the schema right first

Two decisions are cheap now and expensive later — both are made *before* the database exists:

> Generate 3 sample "chemical element" objects with semantic IDs enabled, show me all three,
> then create a schema from them.

- **Semantic IDs** (`generate_semantic_ids=true` on `create_schema_from_sample`) give every
  object a stable, org-scoped identity. Without them each table keys on whatever `is_key`
  property generation happened to pick — a name, a website — which drifts between runs and
  mints duplicate rows. Retrofitting means hand-editing every object. They need an
  organization embedding model.
- **Ownership.** An entity's own parts (isotopes, movements, crew members) are `owned` and
  become child tables. A recurring third party (a laboratory, a publisher, a launch site) must
  **not** be owned: an owned shared entity materializes one private copy per parent instead of
  one row every parent references, and no join undoes that afterwards.

## 1. Connect a database

> Register a database sync named "catalogue" on that schema.

`create_database_sync` returns the database id **and** a `classification_job_id`: registering
starts an LLM pass that proposes each property's SQL contract — which property is the database
key, which columns get indexed, what type each one becomes, which relationships are owned.

> Poll that classification job, then read the schema back and show me the proposed keys, types
> and owned relationships.

`get_job_status` → `get_schema`. This is the review that matters:

- a property whose proposed type cannot hold its own values (a year as text, a count as text);
- a **shared** entity the pass stamped `owned` — the copy-per-parent trap above;
- a declared `is_key` property demoted to something else.

Fix what's wrong with `update_schema` (no LLM call, no cost), or re-run the whole pass with
`classify_database_model`.

## 2. Publish

> Looks right — publish the schema.

`publish_schema`. Linking never publishes: until this call the schema is *unpublished* — no
snapshot, no deltas, no rows. Publishing emits the DDL and starts the feed. It is also where a
multilingual database key locks its language.

From here on, every structural edit publishes the same way: the diff is compared against what
each database has actually shipped and ships as **additive DDL** (applied silently) or a
**confirmed transform migration** (a re-key, a type change, a renamed column — held until you
confirm). You never hand-write an `ALTER`.

## 3. Pair the sync client

> Issue a sync credential for that database.

`create_database_credential` returns a pairing token. Then, next to your database:

```bash
curl -fsSL https://entityenricher.ai/install-eedatabase.sh | sh
ee-database pair --server https://entityenricher.ai <refresh-token>
ee-database run --dsn "postgres://me:secret@localhost:5432/catalogue" --create-missing
```

The DSN never leaves that machine — Entity Enricher holds no credential to your database, and
the client connects outward over WSS. `--create-missing` lets it create the database itself;
add `--admin-dsn` when even the role does not exist yet. First run bootstraps from the `.sql`
snapshot, then it applies leased delta batches and acknowledges them.

A client that can run commands (Claude Code) does all of this for you. Everywhere else, paste
the three lines.

## 4. Enrich, and read what actually landed

> Enrich Gold and Iron against that schema in English and French.

Every enrichment response carries a `database` block — read it instead of guessing:

| Field | Meaning |
|---|---|
| `status` | `saved`, `partial` (the entity landed but some rows were dropped), `rejected` |
| `reason` / `missing_fields` | The unfilled non-nullable path behind the rejection or each dropped item |
| `entity_keys` | The **stored** key column values — correlate rows by these, not by your input text, which the model may canonicalize |
| `key_collisions`, `shared_entity_conflicts`, `skipped_items` | Rows dropped or overwritten silently |
| `identity_merges` | Objects whose semantic ID resolved to a concept minted from *different* text. With an empty `json_path` it is the enriched entity itself: this run took over that row and overwrote it, so check the two texts really name the same thing |

Nothing re-sends dropped rows: fix the schema, re-enrich. A `rejected` entity never reaches
your database at all — waiting for its rows is waiting forever.

## 5. Verify

> Show me the current entity state for that schema.

`list_entity_states` browses the merged, non-null-wins rows the entity layer holds — the same
rows your replica mirrors — so you can check the result without querying your own database.
`list_database_syncs` shows the pending delta count per database: it should drain to zero
shortly after each enrichment.

## Applying the feed without the CLI

When no client can run next to the database, a chat (or an n8n / Make workflow) can apply the
feed itself with `fetch_database_deltas` → your SQL → `ack_database_deltas`. Two rules:

1. **Apply a whole window in one transaction.** All deltas of one enrichment share a batch and
   the projected tables carry deferred foreign keys; a window never splits a batch, so "one
   fetch = one transaction" is the correct unit.
2. **Acknowledge only after the apply succeeded** — an ack on a failed apply loses those deltas
   permanently. Fetch without `claim` for a harmless, replayable read.

Deltas of `kind: "schema"` are DDL migrations and arrive **before** the data rows that need
them — apply them in order, never filter them out.

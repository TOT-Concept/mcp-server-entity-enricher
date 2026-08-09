# Claude Code

Ways to register the Entity Enricher MCP server in [Claude Code](https://claude.com/claude-code),
and what to ask it once it's connected.

## Option 1 — OAuth (recommended)

```bash
claude mcp add --transport http entity-enricher https://entityenricher.ai/api/mcp/
```

Then in a session:

```
> /mcp
```

Select **entity-enricher → Authenticate**. Your browser opens the Entity Enricher consent
screen — sign in if needed and click **Authorize**. The connection acts with your own role;
revoke it anytime under **Settings → API Keys → Connected Apps** in the web UI.

No key to create, nothing secret in your config. The checked-in [`.mcp.json`](./.mcp.json)
is the project-scoped equivalent (commit-safe: it contains only the URL).

## Option 2 — API key (headless / CI)

For non-interactive environments where the OAuth browser flow isn't available, create an
`ent_…` key in the web UI (**Settings → API Keys**) and pass it as a header:

```bash
claude mcp add --transport http entity-enricher https://entityenricher.ai/api/mcp/ \
  --header "X-API-Key: ent_your_key_here"
```

Or in a project `.mcp.json`, keeping the key out of version control via an environment
variable:

```json
{
  "mcpServers": {
    "entity-enricher": {
      "type": "http",
      "url": "https://entityenricher.ai/api/mcp/",
      "headers": {
        "X-API-Key": "${ENTITY_ENRICHER_API_KEY}"
      }
    }
  }
}
```

Then export `ENTITY_ENRICHER_API_KEY=ent_...` in your shell profile.

## Verify

```
> /mcp
```

should list `entity-enricher` as connected. Try:

```
> List my Entity Enricher schemas
```

---

# Examples

Ask in plain language — Claude picks the tools. What makes Claude Code different from the
other clients is that it can also run things on your machine: install the sync client, apply
the SQL, query the resulting database, and write the code around it.

## 1 · Author a schema from a sample

```
> Generate 3 sample "chemical element" objects with Entity Enricher, show me all three,
> then turn them into a saved schema.
```

`generate_sample` → review → `create_schema_from_sample`. Three samples of the same type are
worth more than one: a field missing or null in **any** of them becomes `nullable` instead of
required, and the distinct observed values seed the property examples.

Two things to settle **before** generating, because retrofitting them means editing every
object by hand:

- **Semantic IDs** (`generate_semantic_ids=true`) if this schema will ever feed a database —
  without them, tables key on whatever `is_key` property generation happened to pick.
- **What is a scalar and what is its own object.** A value that names a real-world thing other
  records will also reference (a manufacturer, a laboratory, an author) is a free-text column
  as a string, and a table of its own — joinable, deduplicated — as a nested object. The sample
  is where that is decided.

Depth: [recipes/schema-from-sample.md](../recipes/schema-from-sample.md).

## 2 · Use a file from your repo as the source

```
> Upload ./docs/spec/sensor-datasheet.pdf to Entity Enricher and generate a sample entity
> from it, then create a schema.
```

```
> Enrich "Novartis AG" against the Pharma Company schema using ./reports/annual-2025.pdf
> as source material, in English and French.
```

`upload_attachment` returns a `mode`: `inline_text` (server-extracted text, any model) or
`binary` (original bytes — the model must accept them; with no model pinned, selection narrows
to capable ones). Images always arrive as `binary`, and in sample generation an attachment
switches the tool into *source mode*: it describes what is visible, it does not invent.

## 3 · Design a database from the schema and sync it locally

The one Claude Code can carry end to end, because the last mile is a command on your machine.

```
> Register a database sync named "catalogue" on the Movie schema, wait for the model
> classification to finish, and show me what it proposed for the key columns, the types and
> the owned relationships.
```

`create_database_sync` returns a `classification_job_id` — Claude polls it with
`get_job_status`, then reads the schema back. This is the review that matters: an entity's own
parts should be **owned** (they become child tables), while a recurring third party must **not**
be — an owned shared entity materializes one private copy per parent instead of one row every
parent references, and no join undoes that later. Fix anything wrong with `update_schema`.

```
> Looks right — publish it.
```

`publish_schema`. A linked-but-unpublished schema queues nothing: publishing is what emits the
DDL and starts the feed. Later structural edits publish the same way, and ship as additive DDL
or a confirmed transform migration — you never hand-write an `ALTER`.

```
> Issue a sync credential and set the client up against my local Postgres.
```

`create_database_credential` returns the pairing token; Claude runs the client for you (it will
ask before each command):

```bash
curl -fsSL https://entityenricher.ai/install-eedatabase.sh | sh
ee-database pair --server https://entityenricher.ai <refresh-token>
ee-database run --dsn "postgres://me:secret@localhost:5432/catalogue" --create-missing
```

The DSN never leaves your machine — Entity Enricher holds no credential to your database, and
the client connects outward over WSS. The installer verifies the binary's Sigstore signature
against the release workflow's identity before making it executable.

```
> Enrich Inception and Heat against that schema, then show me what landed in the catalogue
> database and how the movie ↔ studio relationship was modelled.
```

Claude reads each enrichment's `database` block — `status` is `saved`, `partial` (the entity
landed but some rows were dropped) or `rejected`, with `missing_fields` naming the schema gap —
then queries your local Postgres directly with `psql`. Nothing re-sends dropped rows: fix the
schema, re-enrich.

No replica yet? `list_entity_states` browses the same merged rows server-side.

## 4 · Batch enrichment

```
> Batch-enrich these 20 companies from ./data/targets.csv against the Company schema in
> English and French, then list the ones that failed with their error.
```

`start_batch_enrichment` returns `{job_id, total}` immediately — Claude polls `get_job_status`
until a terminal status and reads the outputs with `list_records(job_id=…)`. Ask for the
failures explicitly: a per-entity `error_code` (`model_retired`, `rate_limited`,
`context_length_exceeded`, `provider_timeout`) says whether retrying is worth it.

Depth: [recipes/batch-enrichment.md](../recipes/batch-enrichment.md).

## 5 · One-shot, in a script or CI

```bash
claude -p "Enrich the entity in ./entity.json against the Company schema and write the
result to ./enriched.json" \
  --mcp-config .mcp.json \
  --allowedTools "mcp__entity-enricher__list_schemas,mcp__entity-enricher__enrich_entity,Write"
```

Use the API-key connection here (Option 2) — the OAuth browser flow has nobody to click it.
MCP tools are allowlisted as `mcp__<server-name>__<tool-name>`, so the run stays scoped to the
tools you named.

## Tool permissions

`/mcp` lists the server's tools; `/permissions` is where you pre-approve the ones you use
constantly. Reasonable defaults for a coding session:

| Pattern | Why |
|---|---|
| `mcp__entity-enricher__list_*`, `mcp__entity-enricher__get_*` | Read-only, safe to auto-approve |
| `mcp__entity-enricher__enrich_entity` | Costs credits — approve per call, or allow once you trust the schema |
| `mcp__entity-enricher__delete_*`, `publish_schema`, `create_database_sync` | Leave on ask: they change what your database looks like |

## More recipes

| Recipe | Covers |
|---|---|
| [Schema from sample](../recipes/schema-from-sample.md) | sample → schema → refine → first enrichment |
| [Batch enrichment](../recipes/batch-enrichment.md) | entity lists, external APIs, async polling, partial-failure retry |
| [Model benchmark](../recipes/model-benchmark.md) | scenarios, gold references, auto-scored model comparison |

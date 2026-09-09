# Entity Enricher MCP Server

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server for
[Entity Enricher](https://entityenricher.ai), available at
`https://entityenricher.ai/api/mcp/` (Streamable HTTP).

From an MCP-compatible client you can:

- Design reusable schemas from sample data or documents and edit properties directly.
- Enrich single entities or lists, with multilingual fields and multiple models.
- Fuse results and recover failed expertise domains without repeating successful work.
- Resolve recurring objects to semantic identities and curate their aliases or uncertain matches.
- Derive relational tables and migrations for your PostgreSQL, MySQL or SQLite database.
- Benchmark enrichment, sample generation and schema generation on your own tasks.

Schema validation and model agreement do not establish factual truth or freshness.
Inspect the actual sources, failures and partial outcomes. A successful generation,
entity-layer admission and application on your external replica are distinct outcomes.

The MCP server needs no local installation. Optional database delivery uses
[ee-database](https://github.com/TOT-Concept/ee-database) on your replica host; its DSN stays
there. Managed hosts can provision automatically; manual pairing is also supported.

## Quickstart

### Option 1 — OAuth (recommended)

For claude.ai, Claude Code, Cursor, and any MCP client that implements the standard OAuth
flow. **No API key to create or paste** — the client discovers the authorization server
automatically, your browser opens the Entity Enricher consent screen, and the connection acts
on your behalf with your own role. Revoke it anytime under **Settings → API Keys → Connected
Apps**.

<details open>
<summary><strong>Claude Code</strong></summary>

```bash
claude mcp add --transport http entity-enricher https://entityenricher.ai/api/mcp/
```

Then run `/mcp` in a session and pick **Authenticate** — your browser opens the consent page.
More options (project `.mcp.json`, API-key fallback): [examples/claude-code/](examples/claude-code/)
</details>

<details>
<summary><strong>claude.ai</strong></summary>

**Settings → Connectors → Add custom connector** with URL
`https://entityenricher.ai/api/mcp/`, then click **Authorize** on the consent screen.
Walkthrough: [examples/claude-ai-remote.md](examples/claude-ai-remote.md)
</details>

<details>
<summary><strong>Cursor / other OAuth-capable clients</strong></summary>

Register the URL with no headers and the client prompts you to sign in:
[examples/cursor/mcp.json](examples/cursor/mcp.json)
</details>

### Option 2 — API key (static JSON configuration)

For clients configured via a JSON file rather than an interactive sign-in (Claude Desktop,
Continue, Zed) — and for headless/CI use.

1. In the [Entity Enricher web UI](https://entityenricher.ai): **Settings → API Keys → New
   organization access key**. Pick a role — operator (read-mostly), editor (create/edit
   schemas), or owner (full control, required for benchmarks). Copy the `ent_…` value; it's
   only shown once.
2. For **Claude Desktop**, edit `~/Library/Application Support/Claude/claude_desktop_config.json`
   (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

   ```json
   {
     "mcpServers": {
       "entity-enricher": {
         "url": "https://entityenricher.ai/api/mcp/",
         "headers": { "X-API-Key": "ent_your_key_here" }
       }
     }
   }
   ```

   Restart Claude Desktop. Full file: [examples/claude-desktop/](examples/claude-desktop/)

### Try it

> List my Entity Enricher schemas, then enrich "Sanofi" against the pharmaceutical company
> schema in English and French.

The client discovers the tools, reads the selected schema and returns the result with a link.
Automatic model selection is available; consequential choices are reviewed when needed.

## Guides and tool descriptions

Server instructions explain the workflows. Tool descriptions explain individual calls,
including preconditions, costs and consequential effects. Detailed modeling, recovery and
migration guidance is loaded only when needed.

Use MCP `resources/list` to discover the guide index (`enricher://docs`) and each guide,
then `resources/read` on the desired URI. Guides do not run a model. Clients decide how
resources enter model context; reading them is not guaranteed to be token-free.

The following public recipes are generated from the **same packaged Markdown** the server
serves. Edit the source guides in the main repository, not these generated copies.

| Guide | MCP resource |
|---|---|
| [Schema from samples](examples/recipes/schema-from-sample.md) | `enricher://docs/schema-from-sample` |
| [Schema format and editing](examples/recipes/schema-reference.md) | `enricher://docs/schema-reference` |
| [Documents](examples/recipes/documents.md) | `enricher://docs/documents` |
| [Enrichment, fusion and recovery](examples/recipes/enrichment-and-fusion.md) | `enricher://docs/enrichment-and-fusion` |
| [Batch enrichment](examples/recipes/batch-enrichment.md) | `enricher://docs/batch-enrichment` |
| [Benchmarks](examples/recipes/model-benchmark.md) | `enricher://docs/model-benchmark` |
| [Database sync](examples/recipes/database-sync.md) | `enricher://docs/database-sync` |
| [Semantic identities](examples/recipes/semantic-ids.md) | `enricher://docs/semantic-ids` |

If a client cannot read MCP resources, use these public links. The guides complement
individual tool contracts; ordinary calls do not require reading them all.

## Tools

<!-- TOOL_TABLE_START — generated by backend/scripts/generate_mcp_tool_table.py; do not edit by hand -->
**57 tools**, spanning the full schema-authoring and enrichment surface:

| Category | Tool | Description |
|---|---|---|
| Discovery | `list_models` | List available model keys, nominal capabilities, languages, strategies, auto-selected defaults and organization profile_limits. |
| Schemas | `generate_sample` | Generate editable sample JSON from a free-text request for schema authoring. |
| Schemas | `list_schemas` | List saved schemas in your organization, pinned first. |
| Schemas | `get_schema` | Read a saved schema with its properties, annotations and input_contract. |
| Schemas | `create_schema_from_sample` | Generate and auto-save a schema from reviewed samples, returning schema_id, schema content and record links. |
| Schemas | `save_schema` | Save a directly authored schema and return its ID and link. |
| Schemas | `update_schema` | Edit a saved schema's metadata or replace its full schema_content without an LLM call. |
| Schemas | `get_schema_part` | Read only the schema fragment needed for an edit. |
| Schemas | `get_enum_candidates` | List observed values outside each open enum's current vocabulary, with counts from recent enrichment records. |
| Schemas | `update_schema_property` | Edit or remove one property by path without replacing the full schema. |
| Schemas | `add_schema_property` | Add a property under the root (parent_path=''), an object path or '$defs.X'. |
| Schemas | `move_schema_property` | Move one property into the root, an object path or '$defs.X', preserving its flags and expertise. |
| Schemas | `resolve_unify_proposal` | Resolve one pending entity-type unification proposal from get_schema. |
| Schemas | `publish_schema` | Publish a database-linked schema's working copy as the contract used by enrichment and replicas. |
| Schemas | `delete_schema` | Soft-delete a saved schema by UUID. |
| Schemas | `analyze_sample` | Analyze sample property ambiguity and relationship identity scoping before schema generation. |
| Schemas | `analyze_schema` | Analyze a saved schema's property ambiguity and relationship identity scoping, writing annotations to the schema. |
| Enrichment & fusion | `start_batch_enrichment` | Start billed asynchronous enrichment of an entity list against exactly one of schema_id or target_schema. |
| Enrichment & fusion | `fetch_entities` | Fetch entities from an external REST API using a server-side GET. |
| Enrichment & fusion | `enrich_entity` | Enrich one entity against exactly one of schema_id or target_schema, returning structured output, record_id, costs and any database outcome. |
| Enrichment & fusion | `retry_expertises` | Retry only an existing record's failed expertise domains, then update its output and attempt the run's fusion/synchronization. |
| Enrichment & fusion | `merge_records` | Fuse two or more records of the same entity into a new arbitration record. |
| Job control | `get_job_status` | Read a job's status, progress and compact terminal summary with persisted record IDs. |
| Job control | `cancel_job` | Request cancellation of a pending, running or paused LLM job. |
| Job control | `answer_job_question` | Resume a paused job with answers to the questions returned under pause. |
| Records & stats | `list_records` | List compact, paginated records in your organization, most recent first. |
| Records & stats | `get_record` | Read one persisted record's structured_output, entity_input_data, validation errors, expertise verdicts and metrics. |
| Records & stats | `get_stats` | Read organization-wide record totals, success rate, tokens and cost summary. |
| Benchmarks | `list_benchmark_scenarios` | List compact benchmark scenario summaries and total. |
| Benchmarks | `get_benchmark_scenario` | Read one benchmark scenario with per-model quality, cost and speed results. |
| Benchmarks | `get_benchmark_scenario_results` | Filter, rank and limit a scenario's per-model benchmark results. |
| Benchmarks | `create_benchmark_scenario` | Create a reusable benchmark with a mandatory scoring judge. |
| Benchmarks | `update_benchmark_scenario` | Edit a benchmark's test definition or scoring configuration. |
| Benchmarks | `set_benchmark_reference` | Save the gold reference for an enrichment or schema-generation benchmark. |
| Benchmarks | `delete_benchmark_scenario` | Delete a benchmark scenario and its stored results. |
| Benchmarks | `run_benchmark` | Start billed asynchronous execution and scoring of a benchmark. |
| Attachments | `upload_attachment` | Upload base64 file bytes as reusable source material; returns id and requires_capability. |
| Attachments | `delete_attachment` | Permanently delete an attachment in your organization, including its stored file. |
| Database Sync | `list_database_syncs` | List a saved schema's database registrations, linked schemas, options and sync hosts. |
| Database Sync | `list_entity_states` | Browse a schema's current merged entity rows, not per-run records. |
| Database Sync | `create_database_sync` | Register a saved schema for relational synchronization to PostgreSQL, MySQL or SQLite. |
| Database Sync | `assign_sync_host` | Assign or clear the host provisioning a database sync. |
| Database Sync | `classify_database_model` | Start a billed analysis proposing database keys, SQL types, indexes and relationship ownership on a linked schema. |
| Database Sync | `delete_database_sync` | Delete a database registration and its queued deltas, stopping its feed. |
| Database Sync | `create_database_credential` | Issue a one-time sync-client credential and install/pair/run command suggestions. |
| Database Sync | `fetch_database_deltas` | Read the next ordered window of SQL deltas and canonical payloads for a database sync. |
| Database Sync | `ack_database_deltas` | Acknowledge every delta through up_to_id after successful application, releasing its lease. |
| Database Sync | `sync_records_to_database` | Validate and inject stored or supplied enrichment output into the entity layer and linked syncs. |
| Semantic IDs | `list_semantic_concepts` | Browse organization concepts with aliases, usage counts and type/model facets. |
| Semantic IDs | `get_semantic_concept` | Read one concept's aliases, identity source keys, linked records and nearest neighbors within its own type/model slice. |
| Semantic IDs | `probe_semantic_concept` | Preview identity resolution without adding a concept or increasing its usage. |
| Semantic IDs | `add_semantic_concept` | Add an identity concept at zero usage, or add text as an alias using alias_of. |
| Semantic IDs | `update_concept_alias` | Remove or promote a concept alias using alias IDs from get_semantic_concept. |
| Semantic IDs | `import_semantic_concepts` | Resolve 1..1000 texts against one concept type. |
| Semantic IDs | `merge_semantic_concepts` | Merge a loser concept into a winner. |
| Semantic IDs | `delete_semantic_concepts` | Delete concepts selected by ids, concept_types or unused_only. |
| Semantic IDs | `migrate_semantic_embeddings` | Inspect or migrate the organization's concept embedding space. |
<!-- TOOL_TABLE_END -->

Tool signatures are the callable contract. The wrappers share backend services, but do not
expose every REST/UI option. Schema mutations require editor; benchmark mutations/runs
require owner plus a benchmark-enabled plan. Database registration/credentials require
owner plus a sync-enabled plan. See each tool for its requirements.

## Data resources

| Resource template | Meaning |
|---|---|
| `enricher://schemas/{schema_id}` | Schema working copy as Markdown. For a linked published contract use `get_schema(version="published")`. |
| `enricher://records/{record_id}` | Output and metrics as Markdown; `get_record` adds expertise and database-delivery diagnostics. |

## Jobs and errors

Long-running work uses this server's start → poll → fetch interface. Start tools return a
`job_id`; `generate_sample` can already be paused or completed when it returns. Poll
`get_job_status`, answer paused questions through `answer_job_question`, and retrieve records
by `list_records(job_id=...)`. A missing in-memory job is not proof of completion; check for
persisted records. Cancellation does not undo earlier records or database writes.

Most failures return `success: false`, `error_code` and `message`; some older tools return
only `error` or `message`. A successful MCP transport response does not imply successful
work. Inspect classification warnings, failed model legs, and partial/rejected database
outcomes even when an output is present. Detailed recovery is in the enrichment guide.

## Scope and limitations

- The MCP single/batch enrichment tools do not expose web-search activation; sample generation does.
- Batch enrichment has no fixed 100-entity cap, but live quotas/credits can stop remaining work.
- The batch tool has no `database_sync=false` option; the single-entity tool does.
- Record deletion/restoration, detailed cost analytics, benchmark result import/export and some database administration remain in the web app or REST API.
- Schema documents use the supported JSON Schema dialect with Entity Enricher annotations; see the schema reference before authoring one directly.

## Links

- [Public MCP documentation](https://entityenricher.ai/docs/integrations/mcp)
- [REST API reference](https://entityenricher.ai/docs/api)
- [Client configurations](examples/)
- [MCP Registry](https://registry.modelcontextprotocol.io) — `ai.entityenricher/enricher`; manifest: [server.json](server.json)

## About this repository

This public repository contains documentation and client examples. The server is embedded in
the Entity Enricher backend and maintained in the private monorepo; this directory is synced
as a git subtree. Licensed under the [MIT License](LICENSE).

# Examples

Everything you need to connect a client to the Entity Enricher MCP server, and chat
walkthroughs of the flows people actually run.

## Client setup

| Client | File | Auth |
|---|---|---|
| [Claude Code](claude-code/) | [`claude-code/.mcp.json`](claude-code/.mcp.json) | OAuth (`/mcp` → Authenticate), or `X-API-Key` for headless / CI |
| [claude.ai](claude-ai-remote.md) | — (Settings → Connectors) | OAuth |
| [Claude Desktop](claude-desktop/) | [`claude-desktop/claude_desktop_config.json`](claude-desktop/claude_desktop_config.json) | `X-API-Key` |
| [Cursor](cursor/mcp.json) | [`cursor/mcp.json`](cursor/mcp.json) | OAuth |

The endpoint is the same everywhere: `https://entityenricher.ai/api/mcp/` (keep the trailing
slash — the slashless form 307-redirects). Prefer OAuth wherever the client supports it: no key
to create or paste, it acts with your own role, and you revoke it under **Settings → API Keys →
Connected Apps**.

The [Claude Code page](claude-code/) goes further than setup — it has worked examples (schema
authoring, a file from your repo as source, a database designed and synced locally, batch runs,
one-shot CI usage) and a suggested tool-permission allowlist.

## Recipes

Copy-paste chat walkthroughs. They are client-agnostic: the prompts work in any MCP client.

| Recipe | What it covers |
|---|---|
| [Schema from sample](recipes/schema-from-sample.md) | generate a sample → schema → refine → first enrichment |
| [Database sync](recipes/database-sync.md) | schema → designed relational tables → publish → pair `ee-database` → migrations, and how to read what actually landed |
| [Batch enrichment](recipes/batch-enrichment.md) | entity lists, external APIs, async polling, partial-failure retry |
| [Model benchmark](recipes/model-benchmark.md) | scenarios, gold references, auto-scored model comparison |

**Start here:** *Schema from sample* if you have nothing saved yet — every other recipe assumes
a schema exists. Then *Database sync* if the results should live in your own database, because
two of its decisions (semantic IDs, ownership) have to be made back in the schema.

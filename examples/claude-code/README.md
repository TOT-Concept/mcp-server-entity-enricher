# Claude Code

Two ways to register the Entity Enricher MCP server in [Claude Code](https://claude.com/claude-code).

## Option 1 — CLI (user scope)

```bash
claude mcp add --transport http entity-enricher https://entityenricher.ai/api/mcp/ \
  --header "X-API-Key: ent_your_key_here"
```

Add `--scope project` to share the registration with your team via a checked-in `.mcp.json` (don't commit a real key — use an environment variable expansion instead, see below).

## Option 2 — project `.mcp.json`

Copy [`.mcp.json`](./.mcp.json) to your project root. To keep the key out of version control, reference an environment variable:

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

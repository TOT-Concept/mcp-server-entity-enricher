# Claude Code

Ways to register the Entity Enricher MCP server in [Claude Code](https://claude.com/claude-code).

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

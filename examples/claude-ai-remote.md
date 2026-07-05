# claude.ai (remote connector, OAuth)

OAuth 2.1 is the primary way to connect to Entity Enricher: the backend embeds a full
authorization server, and any MCP client implementing the standard auth spec (claude.ai,
Claude Code, Cursor, MCP Inspector) discovers it automatically. **No API key is needed** —
the connection is authorized interactively and acts with your own Entity Enricher role.
This walkthrough covers claude.ai, where OAuth is required for the connector directory.

## Connect

1. In claude.ai go to **Settings → Connectors → Add custom connector**.
2. Enter the server URL:

   ```
   https://entityenricher.ai/api/mcp/
   ```

3. Claude discovers the authorization server automatically (RFC 9728 protected-resource
   metadata), registers itself (dynamic client registration), and opens the Entity Enricher
   consent page.
4. Sign in if needed and click **Authorize**.

That's it — the connector now appears in Claude's tool menu for every chat where you enable it.

## Security model

- Tokens are **audience-bound** to `/api/mcp`, short-lived, and rotate on refresh.
- The connection acts with **your own role** (operator / editor / owner). System admins are
  capped at owner.
- Every token is validated statefully on each request, so revocation is instant.

## Revoke

In the Entity Enricher web UI: **Settings → API Keys → Connected Apps** → revoke the
connection. Live tokens are cut immediately.

# ModelClone MCP server

Remote **Streamable HTTP** MCP for [Claude.ai connectors](https://docs.claude.ai/en/docs/build-with-claude/mcp) and local **stdio** for Cursor / Claude Desktop.

All tools proxy **`/api/v1`** 1:1 (same auth, paths, bodies, and responses as REST).

## Production connector (Claude.ai)

| Field | Value |
|-------|--------|
| **URL** | `https://mcp.modelclone.app/mcp` |
| **Transport** | Streamable HTTP |
| **Auth** | Your integrator key: `Authorization: Bearer mcl_…` or `X-Api-Key: mcl_…` |

DNS: point `mcp.modelclone.app` CNAME to the same Vercel project as `modelclone.app`.

Discovery: `GET https://mcp.modelclone.app/mcp/health`

## Tools

| Tool | REST equivalent |
|------|-----------------|
| `api_v1_request` | Any allowed `METHOD /api/v1{path}` |
| `get_me` | `GET /me` |
| `list_models` | `GET /models` |
| `list_generations` | `GET /generations` |
| `get_generation` | `GET /generations/:id` |
| `get_pricing_generation` | `GET /pricing/generation` |

## Resources

| URI | Content |
|-----|---------|
| `modelclone://v1/route-catalog` | Integrator route inventory JSON |
| `modelclone://v1/base` | Base URL + auth header hints |

## Local stdio (dev)

```bash
cd integrations/mcp-modelclone
npm install
export MODELCLONE_API_KEY=mcl_...
export MODELCLONE_BASE_URL=https://modelclone.app
npm start
```

Claude Desktop / Cursor `mcpServers` config:

```json
{
  "mcpServers": {
    "modelclone": {
      "command": "node",
      "args": ["<repo>/integrations/mcp-modelclone/src/index.mjs"],
      "env": {
        "MODELCLONE_API_KEY": "mcl_...",
        "MODELCLONE_BASE_URL": "https://modelclone.app"
      }
    }
  }
}
```

## Full documentation

**[MODELCLONE_MCP.md](./MODELCLONE_MCP.md)** — complete MCP handbook (tools, auth, Claude connector, recipes).

Source monorepo: `integrations/mcp-modelclone/` (stdio entry). Regenerate merged doc from shards: `node scripts/merge-mcp-doc.mjs`.

## Implementation

- HTTP: `src/routes/mcp.routes.js` mounted at `/mcp` on the main Express app
- Shared logic: `src/lib/mcp/*`
- Route allowlist: `docs/generated/V1_ROUTE_INVENTORY.json` (`integratorProduct: true` only)

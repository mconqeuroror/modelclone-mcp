## Health endpoint

`GET https://mcp.modelclone.app/mcp/health` — **no authentication**.

Example response (truncated):

```json
{
  "name": "modelclone-mcp",
  "version": "2.0.0",
  "transport": "streamable-http",
  "protocol": "https://modelcontextprotocol.io",
  "connectorUrl": "https://mcp.modelclone.app/mcp",
  "auth": "X-Api-Key: mcl_… or Authorization: Bearer mcl_… on every MCP request (Claude connector OAuth field maps here).",
  "docs": "docs/MCP.md",
  "toolGroups": {
    "meta": ["get_me", "get_pricing_generation", "api_v1_request"],
    "generations": ["list_generations", "get_generation", "wait_for_generation"],
    "models": ["list_models", "get_model", "create_model", "delete_model", "models_generate_reference", "models_generate_poses", "models_status"],
    "wizard": ["wizard_look_variants", "wizard_preview_images", "wizard_custom_reference", "wizard_upload_save", "wizard_finalize_poses"],
    "generate": ["generate_recreate", "generate_free", "enhance_prompt", "generate_motion_video", "creator_studio_image", "creator_studio_video"],
    "nsfw": ["nsfw_generate", "nsfw_generate_video", "nsfw_train_lora", "nsfw_training_status", "nsfw_v2_preset", "nsfw_v2_undress", "nsfw_v2_free_prompt"],
    "nsfwVideo": ["nsfw_video_presets", "nsfw_video_create_session", "nsfw_video_get_session", "nsfw_video_session_action"],
    "modelcloneX": ["mcx_config", "mcx_generate", "mcx_status"],
    "img2img": ["img2img_describe", "img2img_generate", "img2img_status"],
    "gptx": ["gptx_conversations", "gptx_send"],
    "tools": ["upscale_image", "synthid_remove", "video_repurpose_generate", "video_repurpose_job"],
    "flows": ["flows_list", "flows_node_types", "flows_get", "flows_create", "flows_run", "flows_run_status"],
    "gallery": ["gallery_feed", "gallery_post", "gallery_like", "gallery_comment"]
  },
  "resources": [
    "modelclone://v1/route-catalog",
    "modelclone://v1/openapi",
    "modelclone://v1/base",
    "modelclone://v1/getting-started"
  ],
  "restBase": "https://modelclone.app/api/v1"
}
```

## Environment variables

### MCP HTTP server (Vercel / Node)

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | Prisma — API key validation |
| `JWT_SECRET` | Yes | App auth (shared server) |
| `MODELCLONE_BASE_URL` | No | Upstream API origin (default `https://modelclone.app`) |
| `API_V2_BASE_URL` | No | Default `v2BaseUrl` in sunset 410 responses |
| `NODE_ENV` | No | `production` enables strict CORS |

### Stdio client (local)

| Variable | Required | Description |
|----------|----------|-------------|
| `MODELCLONE_API_KEY` | Yes | Your `mcl_…` key |
| `MODELCLONE_BASE_URL` | No | Same as above |

## Source file map

| Path | Role |
|------|------|
| `src/routes/mcp.routes.js` | HTTP router, sessions, health, sunset guard |
| `src/lib/mcp/createMcpServer.js` | Tool + resource definitions |
| `src/lib/mcp/apiV1Client.js` | Upstream fetch to `/api/v1` |
| `src/lib/mcp/allowedRoutes.js` | Integrator allowlist |
| `src/lib/mcp/validateApiKey.js` | Key validation |
| `src/lib/mcp/extractApiKey.js` | Header parsing |
| `src/middleware/api-v1-sunset.middleware.js` | V1 sunset 410 guard |
| `integrations/mcp-modelclone/src/index.mjs` | Stdio entry |
| `vercel.json` | `mcp.modelclone.app` → `/api/index.js` |
| `docs/generated/V1_ROUTE_INVENTORY.json` | Allowlist data |

## npm scripts

```bash
node scripts/merge-mcp-doc.mjs    # Build MODELCLONE_MCP.md from sections
cd integrations/mcp-modelclone && npm start   # Stdio server
```

## Related documents

| Doc | Contents |
|-----|----------|
| `docs/mcp/MODELCLONE_MCP.md` | This handbook (merged) |
| `docs/MCP.md` | Product-facing MCP quickstart + tool reference |
| `docs/public-api/30-mcp.md` | Public API doc MCP chapter |
| `docs/public-api/README.md` | Full REST schemas |
| `integrations/mcp-modelclone/README.md` | Short connector README |
| `docs/public-api/04-rate-limits.md` | Rate limit buckets |
| `docs/public-api/05-webhooks.md` | Webhook HMAC |

## Version

MCP server version: **2.0.0** (`createModelcloneMcpServer` / health). Update this appendix when bumping the version in `mcp.routes.js`.

## Quick setup reference

| Client | Config |
|--------|--------|
| Claude.ai | Settings → Connectors → URL `https://mcp.modelclone.app/mcp`, bearer `mcl_…` |
| Claude Code | `claude mcp add --transport http modelclone https://mcp.modelclone.app/mcp --header "Authorization: Bearer mcl_…"` |
| Cursor | `.cursor/mcp.json` → `"url": "https://mcp.modelclone.app/mcp"`, `"Authorization": "Bearer mcl_…"` |
| Claude Desktop | stdio `node …/integrations/mcp-modelclone/src/index.mjs` or HTTP as above |

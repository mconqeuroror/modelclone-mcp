## What this is

ModelClone exposes a **[Model Context Protocol (MCP)](https://modelcontextprotocol.io)** server so AI clients (Claude.ai, Claude Code, Claude Desktop, Cursor) can call the **same integrator API** as HTTP — not a separate product surface.

| Mode | URL / command | Best for |
|------|----------------|----------|
| **Remote (production)** | `https://mcp.modelclone.app/mcp` | Claude.ai, Claude Code, Cursor (HTTP) |
| **Local stdio** | `integrations/mcp-modelclone/` | Offline dev, clients without HTTP MCP |

**Server version:** `2.1.0` (see `GET …/mcp/health`).

## Architecture

```
Claude / Cursor  --MCP-->  mcp.modelclone.app/mcp  --fetch-->  modelclone.app/api/v1
                              (Streamable HTTP)              (your mcl_ key)
```

1. Client sends MCP tool calls with your **`mcl_`** key on every HTTP request.
2. MCP server validates the key (bcrypt lookup, same as REST).
3. **50+ typed tools** cover every product area; **`api_v1_request`** is the escape hatch for any allowed integrator route.
4. **`api_v1_request`** forwards to `https://modelclone.app/api/v1{path}` with `X-Api-Key`.

Implementation lives in:

- `src/routes/mcp.routes.js` — HTTP transport, sessions, CORS, V1 sunset guard
- `src/lib/mcp/*` — tools, allowlist, API client
- `integrations/mcp-modelclone/` — stdio entry (imports shared core)

## What MCP covers (1:1 with integrator REST)

- All routes where `integratorProduct: true` in `docs/generated/V1_ROUTE_INVENTORY.json` (**~100 routes**).
- Same credits, rate limits, async poll model, webhooks (`integrationCallbackUrl` in JSON bodies).
- Resource `modelclone://v1/route-catalog` embeds the filtered inventory JSON.
- **Flow Studio** — typed tools (`flows_*`) plus `/api/v1/flows/*` via `api_v1_request`.
- **NSFW studio & video**, ModelClone-X, img2img, GPT-X, gallery, upscaling, repurposer.

## What MCP does **not** cover

| Excluded | Why |
|----------|-----|
| `/api/v1/admin/*` | Operator dashboard only; API keys get `403` `ADMIN_SESSION_ONLY` |
| Session-only namespaces | Stripe/crypto billing, referrals, DSAR — see `sessionOnlyNamespaces` in route catalog |
| Browser SPA / cookies | Use `mcl_` key, not login cookies |
| Multipart file upload in one tool | Use presign/upload REST (or `api_v1_request` after you have URLs) |

## Session behavior (remote HTTP only)

| Rule | Detail |
|------|--------|
| **Session binding** | Each `mcp-session-id` is tied to the API key that created it. Reusing a session with a different key → `403` "Session bound to a different API key". |
| **Idle expiry** | Sessions pruned after **30 minutes** without traffic (`SESSION_TTL_MS`). Reconnect — account state (generations, models) persists in the DB. |
| **Cold starts** | Sessions live in lambda memory; Vercel cold start drops them. Clients should re-`initialize` on session errors. |

Stdio transport has **no** HTTP session — each IDE process holds one key in `MODELCLONE_API_KEY`.

## V1 sunset (410)

When the `api_v1_sunset` feature flag is **on**, MCP and `/api/v1` traffic authenticated with an `mcl_` key returns **`410 Gone`** with code **`API_V1_SUNSET`** and a **`v2BaseUrl`** pointer. Session-cookie SPA traffic is unaffected. Watch `docs/API_CHANGELOG.md` for cutover dates.

## Tools at a glance

| Group | Examples |
|-------|----------|
| Meta | `get_me`, `get_pricing_generation`, `api_v1_request` |
| Generations | `list_generations`, `get_generation`, `wait_for_generation` |
| Models / wizard | `list_models`, `wizard_finalize_poses`, `models_status` |
| SFW generate | `generate_recreate`, `creator_studio_image`, `generate_motion_video` |
| NSFW | `nsfw_generate`, `nsfw_v2_preset`, `nsfw_video_*` |
| MCX / img2img / GPT-X | `mcx_generate`, `img2img_generate`, `gptx_send` |
| Flows / gallery | `flows_run`, `gallery_feed` |

Full schemas: §07 · REST field reference: `docs/public-api/`.

## Related docs

- **MCP quickstart:** `docs/MCP.md`
- **Integrator REST:** `docs/public-api/README.md`
- **Local stdio:** `integrations/mcp-modelclone/README.md`

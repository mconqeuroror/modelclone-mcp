# ModelClone MCP — complete reference

**Remote URL:** `https://mcp.modelclone.app/mcp`  
**Transport:** Streamable HTTP (production) · stdio (local dev)  
**REST parity:** All tools proxy `https://modelclone.app/api/v1` 1:1 with your `mcl_` key  
**Generated:** 2026-10-07 — edit `docs/mcp/sections/*.md` then `node scripts/merge-mcp-doc.mjs`

> Full HTTP field reference: [`docs/integrator/MODELCLONE_INTEGRATOR_API.md`](./MODELCLONE_INTEGRATOR_API.md). This document covers **MCP access only**.

---

## Table of contents

- [01. Overview and architecture](#01-overview-and-architecture)
- [02. Deployment and DNS](#02-deployment-and-dns)
- [03. Authentication](#03-authentication)
- [04. Streamable HTTP transport](#04-streamable-http-transport)
- [05. Local stdio transport](#05-local-stdio-transport)
- [06. Tool: api_v1_request](#06-tool-api-v1-request)
- [07. Convenience tools](#07-convenience-tools)
- [08. Resources](#08-resources)
- [09. Claude.ai connector](#09-claude-ai-connector)
- [10. Cursor and Claude Desktop](#10-cursor-and-claude-desktop)
- [11. Security and route allowlist](#11-security-and-route-allowlist)
- [12. Troubleshooting](#12-troubleshooting)
- [13. Recipes](#13-recipes)
- [14. Limitations and REST parity](#14-limitations-and-rest-parity)
- [15. Appendix](#15-appendix)

---


---

## 01. Overview and architecture {#01-overview-and-architecture}

### What this is

ModelClone exposes a **[Model Context Protocol (MCP)](https://modelcontextprotocol.io)** server so AI clients (Claude.ai, Claude Code, Claude Desktop, Cursor) can call the **same integrator API** as HTTP — not a separate product surface.

| Mode | URL / command | Best for |
|------|----------------|----------|
| **Remote (production)** | `https://mcp.modelclone.app/mcp` | Claude.ai, Claude Code, Cursor (HTTP) |
| **Local stdio** | `integrations/mcp-modelclone/` | Offline dev, clients without HTTP MCP |

**Server version:** `2.1.0` (see `GET …/mcp/health`).

### Architecture

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

### What MCP covers (1:1 with integrator REST)

- All routes where `integratorProduct: true` in `docs/generated/V1_ROUTE_INVENTORY.json` (**~100 routes**).
- Same credits, rate limits, async poll model, webhooks (`integrationCallbackUrl` in JSON bodies).
- Resource `modelclone://v1/route-catalog` embeds the filtered inventory JSON.
- **Flow Studio** — typed tools (`flows_*`) plus `/api/v1/flows/*` via `api_v1_request`.
- **NSFW studio & video**, ModelClone-X, img2img, GPT-X, gallery, upscaling, repurposer.

### What MCP does **not** cover

| Excluded | Why |
|----------|-----|
| `/api/v1/admin/*` | Operator dashboard only; API keys get `403` `ADMIN_SESSION_ONLY` |
| Session-only namespaces | Stripe/crypto billing, referrals, DSAR — see `sessionOnlyNamespaces` in route catalog |
| Browser SPA / cookies | Use `mcl_` key, not login cookies |
| Multipart file upload in one tool | Use presign/upload REST (or `api_v1_request` after you have URLs) |

### Session behavior (remote HTTP only)

| Rule | Detail |
|------|--------|
| **Session binding** | Each `mcp-session-id` is tied to the API key that created it. Reusing a session with a different key → `403` "Session bound to a different API key". |
| **Idle expiry** | Sessions pruned after **30 minutes** without traffic (`SESSION_TTL_MS`). Reconnect — account state (generations, models) persists in the DB. |
| **Cold starts** | Sessions live in lambda memory; Vercel cold start drops them. Clients should re-`initialize` on session errors. |

Stdio transport has **no** HTTP session — each IDE process holds one key in `MODELCLONE_API_KEY`.

### V1 sunset (410)

When the `api_v1_sunset` feature flag is **on**, MCP and `/api/v1` traffic authenticated with an `mcl_` key returns **`410 Gone`** with code **`API_V1_SUNSET`** and a **`v2BaseUrl`** pointer. Session-cookie SPA traffic is unaffected. Watch `docs/API_CHANGELOG.md` for cutover dates.

### Tools at a glance

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

### Related docs

- **MCP quickstart:** `docs/MCP.md`
- **Integrator REST:** `docs/public-api/README.md`
- **Local stdio:** `integrations/mcp-modelclone/README.md`


---

## 02. Deployment and DNS {#02-deployment-and-dns}

### Production hosting

| Item | Value |
|------|--------|
| **Hostname** | `mcp.modelclone.app` |
| **Connector endpoint** | `https://mcp.modelclone.app/mcp` |
| **Health** | `GET https://mcp.modelclone.app/mcp/health` (no auth) |
| **App mount** | Express `app.use('/mcp', mcpRoutes)` in `src/server.js` |

### DNS

1. In your DNS provider, add **`mcp.modelclone.app`** as a **CNAME** to your Vercel project (same target as `modelclone.app`).
2. In **Vercel → Project → Domains**, add `mcp.modelclone.app` and wait for **Valid**.

### Vercel routing (critical)

The MCP host must hit the **Express lambda**, not the static SPA.

`vercel.json` must include (first rewrite):

```json
{
  "source": "/:path*",
  "destination": "/api/index.js",
  "has": [{ "type": "host", "value": "mcp.modelclone.app" }]
}
```

**Wrong:** `"destination": "/mcp/:path*"` — serves `index.html` (HTML health check = broken).

**Right:** `"destination": "/api/index.js"` — Express handles `/mcp` and `/mcp/health`.

### Verify after deploy

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

**Success** — JSON with `"name": "modelclone-mcp"`, `"version": "2.1.0"`, `"toolGroups": { … }`, `"restBase": "https://modelclone.app/api/v1"`.

**Failure** — HTML page titled "ModelClone — Cinematic AI Video…" → fix `vercel.json` and redeploy.

### Environment variables (server)

| Variable | Default | Role |
|----------|---------|------|
| `MODELCLONE_BASE_URL` | `https://modelclone.app` | Target for `api_v1_request` upstream |
| `JWT_SECRET` / DB | (required) | API key validation uses Prisma `ApiKey` table |
| `API_V2_BASE_URL` | (optional) | Default `v2BaseUrl` in V1 sunset 410 responses |

MCP does **not** use a server-side `MODELCLONE_API_KEY` — each client supplies their own `mcl_` key per request.

### V1 sunset kill-switch

Controlled by feature flag `api_v1_sunset` (admin). When active, `mcpSunsetGuard` on `POST /mcp` returns **410** before session creation. Health endpoint stays up for discovery.

### Redeploy checklist

- [ ] `vercel.json` host → `/api/index.js`
- [ ] Domain green in Vercel
- [ ] `curl …/mcp/health` returns JSON (not HTML)
- [ ] Claude connector URL = `https://mcp.modelclone.app/mcp` (not `modelclone.app/mcp`)


---

## 03. Authentication {#03-authentication}

### API key format

- Prefix: **`mcl_`**
- Length: **44 characters** total (`mcl_` + 40 random)
- Issued in **Settings → API** (any account) — see integrator auth doc.

### Headers (every MCP HTTP request)

Send **one** of:

| Header | Example |
|--------|---------|
| `X-Api-Key` | `X-Api-Key: mcl_AbCdEf…` |
| `Authorization` | `Authorization: Bearer mcl_AbCdEf…` |
| `Authorization` | `Authorization: ApiKey mcl_AbCdEf…` |

Claude.ai connector: paste the full key in the connector **Bearer / API key** field (maps to the above).

**Do not** send a JWT `Bearer eyJ…` unless you intend session auth on non-MCP routes — integrators should use **`mcl_` only**.

### Validation flow

1. `extractApiKeyFromRequest` reads headers (`src/lib/mcp/extractApiKey.js`).
2. `validateApiKey` bcrypt-compares against `ApiKey` rows (`src/lib/mcp/validateApiKey.js`).
3. Revoked keys and `banLocked` users are rejected.
4. `lastUsedAt` is updated on success (async).

### Error responses (HTTP layer)

Missing key:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Missing API key. Send X-Api-Key: mcl_… or Authorization: Bearer mcl_… (Claude connector auth)."
  },
  "id": null
}
```

Invalid or revoked key:

```json
{
  "jsonrpc": "2.0",
  "error": { "code": -32001, "message": "Invalid or revoked API key" },
  "id": null
}
```

### Eligibility (same as REST)

- Mint keys in Settings — available to every account (no tier gate).
- MCP does **not** bypass plan checks — it proxies your account's real credits and limits.

### Session binding

Streamable HTTP sessions are tied to the API key that created them.

1. First authenticated `POST /mcp` → server creates transport, returns **`mcp-session-id`** header.
2. Follow-up requests send `mcp-session-id` to reuse the transport.
3. If the key on a follow-up request **does not match** the key that created the session:

```json
{
  "jsonrpc": "2.0",
  "error": { "code": -32001, "message": "Session bound to a different API key" },
  "id": null
}
```

HTTP status: **403**. Fix: disconnect and reconnect with one key; do not rotate keys mid-session.

### Idle session expiry

- **`SESSION_TTL_MS` = 30 minutes** — last activity timestamp refreshed on each request.
- Prune job runs every 5 minutes; expired sessions are deleted from memory.
- After expiry, send a fresh `initialize` (clients usually do this automatically).
- **No data loss** — generations, models, and credits live in the database under your account.

### V1 sunset (410)

When feature flag `api_v1_sunset` is **on**, MCP requests with an `mcl_` key receive **410 Gone** before tool execution:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "The ModelClone v1 API has been retired. Please migrate to the v2 API. (v2: https://api-v2.example.com)",
    "data": {
      "success": false,
      "code": "API_V1_SUNSET",
      "message": "The ModelClone v1 API has been retired. Please migrate to the v2 API.",
      "v2BaseUrl": "https://api-v2.example.com"
    }
  },
  "id": null
}
```

Migrate your connector to the **`v2BaseUrl`** from the payload. SPA session traffic on `/api/v1` is **not** blocked by this flag.


---

## 04. Streamable HTTP transport {#04-streamable-http-transport}

### Protocol

Production MCP uses **Streamable HTTP** (`@modelcontextprotocol/sdk/server/streamableHttp.js`).

| | |
|---|---|
| **Endpoint** | `POST https://mcp.modelclone.app/mcp` (also handles session continuation) |
| **Health** | `GET https://mcp.modelclone.app/mcp/health` (no auth) |
| **Spec** | [Model Context Protocol](https://modelcontextprotocol.io) · [Claude MCP connectors](https://docs.claude.ai/en/docs/build-with-claude/mcp) |

### Session lifecycle

1. First `POST /mcp` with valid `mcl_` key → server creates `StreamableHTTPServerTransport` + in-memory session.
2. Response includes **`mcp-session-id`** header (UUID).
3. Follow-up requests send `mcp-session-id` to reuse the transport; **`entry.at`** refreshed on each hit.
4. Sessions expire after **30 minutes** idle (`SESSION_TTL_MS`); pruned every 5 minutes.

#### Session binding

Each session stores the **`apiKey`** that initialized it. A follow-up request with a **different** key but the same `mcp-session-id` is rejected:

- HTTP **403**
- JSON-RPC code **-32001**
- Message: **"Session bound to a different API key"**

Do not swap keys on a live connector without reconnecting.

#### After idle expiry or cold start

| Event | Client action |
|-------|---------------|
| 30 min idle | Re-`initialize`; new `mcp-session-id` issued |
| Vercel cold start | In-memory session gone; same as above |
| Lambda scale-out | Session may not exist on new instance; retry initialize |

Account data (generations, models, credits) is **not** stored in the MCP session — only transport state.

### V1 sunset on `/mcp`

`mcpSunsetGuard` runs before auth when `api_v1_sunset` flag is on. Returns **410** with `API_V1_SUNSET` + `v2BaseUrl` (see §3). Health endpoint is **not** gated.

### Cold starts (Vercel)

Sessions live **in memory** on the lambda instance. After a cold start, clients must **re-initialize** (new `initialize` / new session). Claude connectors usually handle this; custom clients should retry on session errors.

**Future improvement:** Redis-backed session store (not shipped).

### CORS

Allowed browser origins (production):

- `https://claude.ai` and `*.claude.ai`
- `*.anthropic.com`
- `localhost` / `127.0.0.1` (dev)

Other origins are rejected in production (`Origin not allowed for MCP`).

### Allowed hosts (transport)

`StreamableHTTPServerTransport` validates Host:

- `mcp.modelclone.app`
- `modelclone.app`, `www.modelclone.app`
- `localhost`, `127.0.0.1`, `[::1]`

### Headers clients may send

| Header | Purpose |
|--------|---------|
| `Authorization` / `X-Api-Key` | `mcl_` key |
| `mcp-session-id` | Session continuity |
| `mcp-protocol-version` | Protocol negotiation |
| `Content-Type` | `application/json` |
| `Accept` | `application/json, text/event-stream` |

### OPTIONS

`OPTIONS /mcp` and `OPTIONS /mcp/health` return **204** for CORS preflight.


---

## 05. Local stdio transport {#05-local-stdio-transport}

### When to use stdio

| Use stdio | Use remote HTTP |
|-----------|-----------------|
| Cursor / Claude Desktop without HTTP MCP | Claude.ai cloud connector |
| Local dev without DNS | Production shared connector URL |
| Air-gapped or custom agents | No local Node process |

Both use **`createModelcloneMcpServer`** — identical tools and allowlist.

> **Recommended for most users:** remote HTTP at `https://mcp.modelclone.app/mcp` (see §9–§10). Stdio is optional for local dev.

### Install

```bash
cd integrations/mcp-modelclone
npm install
```

Requires **Node 18+**.

### Environment

```bash
export MODELCLONE_API_KEY=mcl_your_key_here
export MODELCLONE_BASE_URL=https://modelclone.app   # optional; default shown
npm start
```

The process speaks **JSON-RPC over stdin/stdout**. Logs/errors go to stderr.

### Claude Desktop (`claude_desktop_config.json`)

**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`  
**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`  
**Linux:** `~/.config/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "modelclone": {
      "command": "node",
      "args": ["C:/Users/YOU/path/to/modelclone/integrations/mcp-modelclone/src/index.mjs"],
      "env": {
        "MODELCLONE_API_KEY": "mcl_YOUR_KEY_HERE",
        "MODELCLONE_BASE_URL": "https://modelclone.app"
      }
    }
  }
}
```

Use **absolute paths**. Restart Claude Desktop after edits.

#### Claude Desktop — HTTP alternative

If your Claude Desktop build supports Streamable HTTP connectors, prefer:

- URL: `https://mcp.modelclone.app/mcp`
- Header: `Authorization: Bearer mcl_YOUR_KEY_HERE`

Same config as Claude.ai (§9) — no local Node process.

### Cursor (stdio)

**Settings → MCP → Add server** or edit `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "modelclone": {
      "command": "node",
      "args": ["C:/Users/YOU/path/to/modelclone/integrations/mcp-modelclone/src/index.mjs"],
      "env": {
        "MODELCLONE_API_KEY": "mcl_YOUR_KEY_HERE",
        "MODELCLONE_BASE_URL": "https://modelclone.app"
      }
    }
  }
}
```

Reload the Cursor window. Confirm tools: `get_me`, `api_v1_request`, …

### Cursor / Claude Code — remote HTTP (recommended)

See §10 for copy-paste HTTP configs (no local Node).

### Troubleshooting

| Symptom | Fix |
|---------|-----|
| `MODELCLONE_API_KEY must be set` | Export `mcl_` key before start |
| Server exits immediately | Run `npm install` in `integrations/mcp-modelclone`; use absolute path in `args` |
| Tools empty in client | Restart IDE; check stderr for import errors |
| 401 on tool calls | Key revoked or typo — mint new key in Settings |
| `410 API_V1_SUNSET` | V1 retired — migrate to `v2BaseUrl` from error (HTTP stdio still hits `/api/v1`) |


---

## 06. Tool: api_v1_request {#06-tool-api-v1-request}

### Purpose

**`api_v1_request`** is the generic escape hatch. It proxies any **integrator-product** route on `/api/v1` with the same method, path, query, and JSON body as REST. Typed tools (e.g. `get_me`, `generate_recreate`) are preferred when available — they are shorter to call and self-documenting.

| | |
|---|---|
| **Tool name** | `api_v1_request` |
| **REST mapping** | Any allowed `METHOD /api/v1{path}` |
| **Description** | Call any integrator-product route 1:1. Allowed paths: resource `modelclone://v1/route-catalog`. Request/response shapes: `modelclone://v1/openapi`. |

### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `method` | `"GET"` \| `"POST"` \| `"PUT"` \| `"PATCH"` \| `"DELETE"` | **Yes** | — | HTTP method |
| `path` | string | **Yes** | — | Path **without** `/api/v1` prefix, e.g. `/generations`, `/nsfw/generate` |
| `query` | object (string/number/boolean values) | No | omitted | Query string key-values |
| `body` | object | No | omitted | JSON body for POST/PUT/PATCH |

### Example MCP tool call

```json
{
  "name": "api_v1_request",
  "arguments": {
    "method": "GET",
    "path": "/me"
  }
}
```

### Example response (success)

All proxy tools return JSON in a single MCP `text` content block:

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/me",
  "url": "https://modelclone.app/api/v1/me",
  "body": {
    "success": true,
    "apiVersion": 1,
    "data": {
      "user": {
        "id": "ckq1x2y3z0000abcd1234efgh",
        "email": "dev@example.com",
        "totalCredits": 870,
        "subscriptionTier": "business",
        "subscriptionStatus": "active"
      },
      "authVia": "api_key"
    }
  }
}
```

### Example response (upstream error)

HTTP errors from REST appear in `httpStatus` and `body` — same as calling REST directly:

```json
{
  "httpStatus": 429,
  "ok": false,
  "method": "POST",
  "path": "/generate/free",
  "url": "https://modelclone.app/api/v1/generate/free",
  "body": {
    "success": false,
    "code": "rate_limited",
    "message": "Too many requests"
  }
}
```

### Example response (allowlist rejection)

Thrown before fetch when the path is not on the integrator product surface:

```json
{
  "ok": false,
  "error": "Route not on integrator product surface: POST /admin/stats. Use resource modelclone://v1/route-catalog."
}
```

### Poll / completion flow

- **Sync routes** (e.g. `GET /me`, `GET /pricing/generation`, `POST /generate/enhance-prompt`): the response in `body` is final — no polling.
- **Async generation submits** (e.g. `POST /generate/recreate`, `POST /nsfw/generate`): read the generation id from `body.generation.id` (or `body.generationId` / `body.generationIds[]` depending on route), then poll with **`wait_for_generation`** or **`get_generation`**. See §07 Generations tools.

### Common errors

| HTTP / shape | When | What to do |
|---|---|---|
| **401** in `httpStatus` | Missing or invalid API key | Send `Authorization: Bearer mcl_…` or `X-Api-Key: mcl_…` on the MCP connection |
| **402** / **403** in `httpStatus`, `body.message` starts with `"Insufficient credits"` | Balance too low for a submit | Call `get_me` for `totalCredits`; call `get_pricing_generation` for costs |
| **404** in `httpStatus` | Resource id not found or belongs to another account | Verify the id; re-submit if expired |
| **429** in `httpStatus` | Rate limit exceeded | Back off and retry; see `docs/public-api/04-rate-limits.md` |
| `{ "ok": false, "error": "Route not on integrator product surface…" }` | Path not on allowlist (billing, admin, session-only) | Use only paths from `modelclone://v1/route-catalog` |

### Credit cost notes

This tool does not charge credits by itself — costs depend on the route you call. Before any generation submit, call **`get_pricing_generation`** for live prices and **`get_me`** to confirm balance.

### Allowlist

Before fetch, `assertIntegratorRouteAllowed` checks `docs/generated/V1_ROUTE_INVENTORY.json` (`integratorProduct: true` only). Admin paths, billing, and internal callbacks are **not** on the allowlist.

### More examples

#### GET /generations (history)

```json
{
  "name": "api_v1_request",
  "arguments": {
    "method": "GET",
    "path": "/generations",
    "query": {
      "status": "completed",
      "limit": 20,
      "includeTotal": true
    }
  }
}
```

Prefer the typed **`list_generations`** tool for the same call.

#### POST creator-studio image (async)

```json
{
  "name": "api_v1_request",
  "arguments": {
    "method": "POST",
    "path": "/generate/creator-studio",
    "body": {
      "prompt": "product photo on white background",
      "generationModel": "nano-banana-pro",
      "aspectRatio": "1:1",
      "numImages": 1
    }
  }
}
```

Poll with **`wait_for_generation`** using `body.generation.id` from the response.

#### POST with integrator webhook

```json
{
  "name": "api_v1_request",
  "arguments": {
    "method": "POST",
    "path": "/generate/free",
    "body": {
      "prompt": "sunset landscape",
      "integrationCallbackUrl": "https://your.app/hooks/modelclone",
      "integratorWebhookSecret": "your-hmac-secret"
    }
  }
}
```

Same webhook fields as REST — see `docs/public-api/05-webhooks.md`.


---

## 07. Convenience tools {#07-convenience-tools}

## Typed tools (full product surface)

As of MCP v2.1.0 the server exposes **226 tools**, including `get_creative_skill`, `wait_for_generation`, and `api_v1_request`, covering every integrator-product workflow. Integrator overview: **`docs/public-api/30-mcp.md`**. Setup entry point: **`docs/MCP.md`**.

| Group | Tools |
|-------|-------|
| Meta & account | `get_me`, `get_pricing_generation`, `confirm_adult`, `get_notifications`, `notifications_mark_read`, `notifications_mark_all_read`, `get_notification_preferences`, `set_notification_preferences`, `get_my_flags`, `get_plans`, `api_v1_request` |
| Generations | `list_generations`, `get_generation`, **`wait_for_generation`**, `generations_batch_delete`, `generations_monthly_stats` |
| Models & wizard | `list_models`, `get_model`, `create_model`, `delete_model`, `models_generate_reference`, `models_generate_poses`, `models_status`, `wizard_*` (5) |
| SFW generate | `generate_image_identity`, `generate_recreate`, `generate_free`, `generate_model_caps`, `list_styles`, `generate_preset_recreate`, `enhance_prompt`, `generate_motion_video`, `generate_video_motion`, `generate_video_directly`, `generate_face_swap_video`, `generate_image_faceswap`, `generate_complete_recreation`, `describe_target`, `extract_frames`, `generate_advanced`, `creator_studio_*` (8) |
| NSFW | `nsfw_*` (18) + `sexting_*` (6) + `nsfw_video_*` (4) |
| ModelClone-X | `mcx_config`, `mcx_generate`, `mcx_status` |
| img2img | `img2img_describe`, `img2img_describe_status`, `img2img_generate`, `img2img_status` |
| GPT-X | `gptx_conversations`, `gptx_send` |
| Tools | `upscale_image`, `synthid_remove`, `video_repurpose_*`, `reformatter_convert`, `reformatter_status` |
| Flow Studio | `flows_*` (10) |
| Gallery & profile | `gallery_*` (7), `set_username` |
| Avatars | `avatars_*` (5) |
| Marketing Studio | `marketing_studio_*`, `marketing_products_*`, `marketing_avatars_*`, `marketing_hooks_list`, `marketing_settings_list`, `marketing_ad_*`, `marketing_static_ad_*`, `marketing_speech_*`, `marketing_production_*`, **`marketing_agent_*` (20 session tools)**, **`get_creative_skill`** |

All tool results are JSON in a single `text` content block.

---

### Meta and account tools

#### `get_me`

| | |
|---|---|
| **Tool name** | `get_me` |
| **REST mapping** | `GET /me` |
| **Description** | Profile, credit balance, and subscription tier. Call **first in every session** to confirm the API key works and you have enough credits. |

##### Input schema

No parameters.

##### Example MCP tool call

```json
{
  "name": "get_me",
  "arguments": {}
}
```

##### Example response

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/me",
  "url": "https://modelclone.app/api/v1/me",
  "body": {
    "success": true,
    "apiVersion": 1,
    "data": {
      "user": {
        "id": "ckq1x2y3z0000abcd1234efgh",
        "email": "dev@example.com",
        "name": "Dev Account",
        "credits": 120,
        "subscriptionCredits": 500,
        "purchasedCredits": 250,
        "totalCredits": 870,
        "subscriptionTier": "business",
        "subscriptionStatus": "active",
        "isVerified": true,
        "onboardingCompleted": true
      },
      "authVia": "api_key"
    }
  }
}
```

##### Poll / completion flow

Synchronous — no polling. Read `body.data.user.totalCredits` before submitting generations.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **404** | Account no longer exists | Contact support |
| **429** | Too many requests | Retry with backoff |

##### Credit cost notes

**Free** — no credits charged. Use with **`get_pricing_generation`** before quoting costs to users.

---

#### `get_pricing_generation`

| | |
|---|---|
| **Tool name** | `get_pricing_generation` |
| **REST mapping** | `GET /pricing/generation` |
| **Description** | Authoritative live credit costs per generation type and option. Prices are server-configurable — never hard-code costs in agents or integrations. |

##### Input schema

No parameters.

##### Example MCP tool call

```json
{
  "name": "get_pricing_generation",
  "arguments": {}
}
```

##### Example response

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/pricing/generation",
  "url": "https://modelclone.app/api/v1/pricing/generation",
  "body": {
    "success": true,
    "pricing": {
      "recreateImage": 10,
      "freePromptImage": 5,
      "motionVideoPerSecond": 9.5,
      "nsfwImage": 6
    },
    "updatedAt": "2026-07-01T12:00:00.000Z",
    "contract": {
      "keys": ["recreateImage", "freePromptImage"],
      "defaults": { "recreateImage": 10 }
    }
  }
}
```

The `contract` object documents what each key in `pricing` means. Re-fetch periodically if you display prices to users.

##### Poll / completion flow

Synchronous — no polling.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **429** | Too many requests | Retry with backoff |
| **500** | Server error loading pricing | Retry with backoff |

##### Credit cost notes

**Free** — returns costs; does not charge credits. This is the **authoritative** source for all generation pricing referenced by other tools.

---

### Generation history and polling tools

Every async submit tool returns a generation id. Use these three tools to list history, poll once, or poll server-side until completion.

#### `list_generations`

| | |
|---|---|
| **Tool name** | `list_generations` |
| **REST mapping** | `GET /generations` |
| **Description** | Paginated generation history across all types, newest first. |

##### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `type` | string | No | omitted | Filter by generation type (e.g. `image`, `video`, `nsfw`, `modelclone-x`, `creator-studio`) |
| `modelId` | string (UUID) | No | omitted | Only generations for one saved model |
| `status` | string | No | omitted | `processing`, `completed`, or `failed`; comma-separate for multiple (e.g. `completed,failed`) |
| `limit` | integer | No | `50` (REST default) | Page size, min 1, max 200 |
| `offset` | integer | No | `0` | Rows to skip, min 0 |
| `includeTotal` | boolean | No | `false` | When `true`, includes `pagination.total` row count |

##### Example MCP tool call

```json
{
  "name": "list_generations",
  "arguments": {
    "status": "completed",
    "limit": 20,
    "offset": 0,
    "includeTotal": true
  }
}
```

##### Example response

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/generations",
  "url": "https://modelclone.app/api/v1/generations?status=completed&limit=20&offset=0&includeTotal=true",
  "body": {
    "success": true,
    "generations": [
      {
        "id": "gen_abc",
        "modelId": "model_123",
        "type": "image",
        "status": "completed",
        "prompt": "golden hour portrait on a rooftop",
        "outputUrl": "https://cdn.modelclone.app/generations/gen_abc.png",
        "errorMessage": null,
        "creditsCost": 6,
        "creditsRefunded": false,
        "createdAt": "2026-07-07T20:00:00.000Z",
        "completedAt": "2026-07-07T20:00:41.000Z"
      }
    ],
    "pagination": { "total": 1342, "limit": 20, "offset": 0 },
    "retention": { "maxCompletedPerModel": 500 }
  }
}
```

##### Poll / completion flow

Synchronous listing — for a single in-flight job, use **`get_generation`** or **`wait_for_generation`**.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **429** | History list rate limit (90/min) | Reduce request frequency |

##### Credit cost notes

**Free** — no credits charged.

---

#### `get_generation`

| | |
|---|---|
| **Tool name** | `get_generation` |
| **REST mapping** | `GET /generations/:id` |
| **Description** | Single generation status and `outputUrl`. Canonical one-shot poll target after any submit. |

##### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `generationId` | string | **Yes** | — | Generation id returned by any submit tool |

##### Example MCP tool call

```json
{
  "name": "get_generation",
  "arguments": {
    "generationId": "gen_abc"
  }
}
```

##### Example response (processing)

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/generations/gen_abc",
  "url": "https://modelclone.app/api/v1/generations/gen_abc",
  "body": {
    "success": true,
    "generation": {
      "id": "gen_abc",
      "type": "image",
      "status": "processing",
      "outputUrl": null,
      "errorMessage": null,
      "creditsCost": 10,
      "createdAt": "2026-07-07T20:00:00.000Z"
    }
  }
}
```

##### Example response (completed)

When `body.generation.status` is `"completed"`, read `body.generation.outputUrl`.

##### Poll / completion flow

1. Submit any generation tool → extract `generationId`.
2. Call **`get_generation`** every 2–5 seconds until `status` is `completed` or `failed`.
3. Prefer **`wait_for_generation`** instead of client-side loops.

Status polls are rate-limited (120/min per account).

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **404** | Id not found or belongs to another account | Verify id from submit response |
| **429** | Poll rate limit exceeded | Use **`wait_for_generation`** or increase interval |

##### Credit cost notes

**Free** — polling does not charge credits. Generation cost was charged at submit time; see **`get_pricing_generation`** for price list.

---

#### `wait_for_generation`

| | |
|---|---|
| **Tool name** | `wait_for_generation` |
| **REST mapping** | Server-side loop over `GET /generations/:id` |
| **Description** | Polls until the generation reaches a terminal status (`completed`/`failed`) or the timeout elapses. **Prefer this after any submit** instead of manual `get_generation` loops. |

##### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `generationId` | string | **Yes** | — | Generation id from any submit tool |
| `timeoutSec` | integer | No | `300` | Max wait time in seconds, min 5, max 570 |
| `intervalSec` | integer | No | `5` | Seconds between polls, min 2, max 30 |

##### Example MCP tool call

```json
{
  "name": "wait_for_generation",
  "arguments": {
    "generationId": "gen_abc",
    "timeoutSec": 180,
    "intervalSec": 5
  }
}
```

##### Example response (completed)

```json
{
  "done": true,
  "status": "completed",
  "generation": {
    "id": "gen_abc",
    "type": "image",
    "status": "completed",
    "outputUrl": "https://cdn.modelclone.app/generations/gen_abc.png",
    "errorMessage": null,
    "creditsCost": 10,
    "creditsRefunded": false,
    "completedAt": "2026-07-07T20:00:41.000Z"
  }
}
```

##### Example response (failed)

```json
{
  "done": true,
  "status": "failed",
  "generation": {
    "id": "gen_abc",
    "status": "failed",
    "outputUrl": null,
    "errorMessage": "Generation failed — credits refunded",
    "creditsRefunded": true
  }
}
```

##### Example response (timeout — still processing)

```json
{
  "done": false,
  "timedOut": true,
  "hint": "Still processing — call wait_for_generation or get_generation again.",
  "last": {
    "success": true,
    "generation": {
      "id": "gen_abc",
      "status": "processing"
    }
  }
}
```

##### Example response (not found)

```json
{
  "done": false,
  "error": "Generation not found",
  "last": {
    "httpStatus": 404,
    "ok": false,
    "body": { "success": false, "message": "Generation not found" }
  }
}
```

##### Poll / completion flow

1. Submit → get `generationId`.
2. Call **`wait_for_generation`** once with an appropriate `timeoutSec` (videos may need 300+).
3. If `timedOut: true`, call again with the same id to keep waiting.
4. When `done: true` and `status: "completed"`, download `generation.outputUrl`.

Unlike other typed tools, this tool's response is **not** wrapped in `httpStatus`/`body` — it returns the poll result directly.

##### Common errors

| Shape | When | What to do |
|---|---|---|
| `{ "done": false, "error": "Generation not found" }` | **404** on poll | Verify generation id |
| `{ "done": false, "timedOut": true, … }` | Job still running after `timeoutSec` | Call again or increase timeout |
| `{ "ok": false, "error": "…" }` | Network or unexpected failure | Retry; check MCP connectivity |
| Underlying **401** / **429** on poll requests | Auth or rate limit during wait | Fix key; reduce poll frequency |

##### Credit cost notes

**Free** — waiting does not charge credits. Submit cost is in `generation.creditsCost`; compare against **`get_pricing_generation`** before submit.

---

#### Terminal generation states

| status | Meaning |
|--------|---------|
| `completed` | `outputUrl` set — download the asset |
| `failed` | `errorMessage` set; credits refunded when applicable (`creditsRefunded: true`) |

Treat `completed` and `failed` as terminal. Stuck jobs are auto-failed by a server watchdog.

---

### Models and wizard tools

REST reference: [`10-models-and-wizard.md`](../../public-api/10-models-and-wizard.md). Model creation is **asynchronous** in the final pose step: pose/upload endpoints return **202** with `model.status: "processing"`. Poll with **`models_status`** every 3–5s until `ready` or `failed`. Confirm capacity with **`list_models`** (`canCreateMore`) before creating.

Live credit costs: **`get_pricing_generation`**. Defaults below — treat the pricing response as authoritative.

| Tool | Pricing key | Default credits |
|------|-------------|-----------------|
| `models_generate_reference` | `modelStep1Reference` | 150 |
| `models_generate_poses` | `modelStep2Poses` | 750 |
| `wizard_custom_reference` (`regenerate: true`) | `wizardCustomRegenerate` | 10 |
| All other model/wizard tools in this section | — | 0 |

#### Creation paths

| Path | Tools (in order) | Poll |
|------|------------------|------|
| Wizard — niche | `wizard_look_variants` → `wizard_preview_images` → `wizard_finalize_poses` | `models_status` |
| Wizard — custom text | `wizard_custom_reference` → `wizard_finalize_poses` | `models_status` |
| Wizard — upload | `wizard_upload_save` | `models_status` |
| Classic two-step | `models_generate_reference` → `models_generate_poses` | `models_status` |
| Direct URLs | `create_model` | — (sync) |

> **NSFW eligibility:** models from user uploads (`create_model`, `wizard_upload_save`) are not AI-generated and **cannot** use NSFW features.

#### `list_models`

| | |
|---|---|
| **Tool name** | `list_models` |
| **REST mapping** | `GET /models` |
| **Description** | Saved AI models for this account plus plan limit metadata. |

##### Input schema

No parameters.

##### Example MCP tool call

```json
{ "name": "list_models", "arguments": {} }
```

##### Poll / completion flow

Synchronous. Read `body.models[]`, `body.count`, `body.limit`, and `body.canCreateMore` before starting creation.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Invalid API key | Fix MCP auth |
| **429** | Rate limit | Back off and retry |

##### Credit cost notes

**Free** — no credits charged.

---

#### `get_model`

| | |
|---|---|
| **Tool name** | `get_model` |
| **REST mapping** | `GET /models/:id` |
| **Description** | One saved model — reference photos, `status`, appearance, voice metadata. |

##### Input schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string (UUID) | **Yes** | Model id from `list_models` or creation response |

##### Example MCP tool call

```json
{ "name": "get_model", "arguments": { "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
```

##### Poll / completion flow

Synchronous. When `body.model.status` is `processing`, poll with **`models_status`** instead.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Invalid API key | Fix MCP auth |
| **404** | Model not found | Verify id |
| **429** | Rate limit | Back off |

##### Credit cost notes

**Free**.

---

#### `create_model`

| | |
|---|---|
| **Tool name** | `create_model` |
| **REST mapping** | `POST /models` |
| **Description** | Save a model from three hosted photo URLs (upload via `POST /upload` first). Synchronous — no generation job. |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `name` | string | **Yes** | Display name (unique per account) |
| `photo1Url`, `photo2Url`, `photo3Url` | string (URL) | **Yes** | Selfie, portrait, full-body references |
| `savedAppearance` | object | No | Appearance attribute map |

##### Example MCP tool call

```json
{
  "name": "create_model",
  "arguments": {
    "body": {
      "name": "Jordan",
      "photo1Url": "https://cdn.example.com/selfie.jpg",
      "photo2Url": "https://cdn.example.com/portrait.jpg",
      "photo3Url": "https://cdn.example.com/fullbody.jpg"
    }
  }
}
```

##### Poll / completion flow

Synchronous — model is `ready` immediately. Use for direct URL saves; prefer wizard/classic flows for AI-generated identities.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **400** | Missing fields or invalid URLs | Fix body; upload photos first |
| **409** | Name collision or plan limit | Rename or delete a model |
| **429** | Rate limit | Back off |

##### Credit cost notes

**Free**. User-uploaded models are **not** NSFW-eligible.

---

#### `delete_model`

| | |
|---|---|
| **Tool name** | `delete_model` |
| **REST mapping** | `DELETE /models/:id` |
| **Description** | Delete a model and related assets. |

##### Input schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string (UUID) | **Yes** | Model to delete |

##### Example MCP tool call

```json
{ "name": "delete_model", "arguments": { "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
```

##### Poll / completion flow

Synchronous.

##### Common errors

| HTTP | When | What to do |
|---|---|---|
| **404** | Model not found | Verify id |
| **409** | Generations still in progress for this model | Wait for jobs to finish |

##### Credit cost notes

**Free**.

---

#### `models_generate_reference`

| | |
|---|---|
| **Tool name** | `models_generate_reference` |
| **REST mapping** | `POST /models/generate-reference` |
| **Description** | Classic phase 1 — generate a reference face from appearance chips. **Synchronous** (200). |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age` | string/number | **Yes** | Age 18–120 |
| Appearance chip fields | string | **Yes** | Ethnicity, hair, skin, eyes, face, body — see REST doc |
| `referencePrompt` | string | No | Creative hint |
| `regenerate` | boolean | No | Force a new reference |

##### Example MCP tool call

```json
{
  "name": "models_generate_reference",
  "arguments": {
    "body": {
      "gender": "female",
      "age": "25",
      "referencePrompt": "soft natural makeup",
      "ethnicity": "Latina",
      "hairColor": "Dark Brown",
      "hairType": "Wavy",
      "skinTone": "Medium",
      "eyeColor": "Brown",
      "eyeShape": "Almond",
      "faceShape": "Oval",
      "noseShape": "Straight",
      "lipSize": "Medium",
      "bodyType": "Athletic",
      "height": "Average",
      "breastSize": "Medium",
      "buttSize": "Medium",
      "waist": "Slim",
      "hips": "Medium",
      "tattoos": "None"
    }
  }
}
```

##### Poll / completion flow

Synchronous — read `body.referenceUrl`. Missing appearance categories → **400** with `missing` array. Feed `referenceUrl` into **`models_generate_poses`**.

##### Credit cost notes

Pricing key `modelStep1Reference` (default **150** credits). Confirm via **`get_pricing_generation`**.

---

#### `models_generate_poses`

| | |
|---|---|
| **Tool name** | `models_generate_poses` |
| **REST mapping** | `POST /models/generate-poses` |
| **Description** | Classic phase 2 — create model row + async 3-pose reference set. |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `name` | string | **Yes** | Model display name |
| `referenceUrl` | string (URL) | **Yes** | From `models_generate_reference` |
| `gender`, `age`, appearance fields | — | Recommended | Passed through to generation |
| `posesPrompt`, `outfitType`, `poseStyle` | string | No | Pose styling hints |

##### Example MCP tool call

```json
{
  "name": "models_generate_poses",
  "arguments": {
    "body": {
      "name": "Riley",
      "referenceUrl": "https://cdn.modelclone.app/references/ref-4f2a.jpg",
      "gender": "female",
      "age": "25"
    }
  }
}
```

##### Poll / completion flow

**Async** (202). Poll **`models_status`** with `body.model.id` every 3–5s until `status` is `ready` or `failed`.

##### Credit cost notes

Pricing key `modelStep2Poses` (default **750** credits).

---

#### `models_status`

| | |
|---|---|
| **Tool name** | `models_status` |
| **REST mapping** | `GET /models/status/:id` |
| **Description** | Model creation job status — canonical poll target after async pose/upload/finalize steps. |

##### Input schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string (UUID) | **Yes** | Model id from async create response |

##### Example MCP tool call

```json
{ "name": "models_status", "arguments": { "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
```

##### Poll / completion flow

Poll every 3–5s. Terminal values: `ready` (use model id in generation tools) or `failed` (read `message`).

##### Credit cost notes

**Free** — polling only.

---

#### `wizard_look_variants`

| | |
|---|---|
| **Tool name** | `wizard_look_variants` |
| **REST mapping** | `POST /wizard/niche/look-variants` |
| **Description** | Wizard step 1 — four niche look palettes for a target persona. **Free, synchronous.** |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age`, `nicheName` | — | **Yes** | Target niche (e.g. Fitness) |
| `nicheId`, `nicheBio` | string | No | Optional niche metadata |
| `ethnicity`, `hairColor`, `eyeColor`, `bodyType` | string | No | Seed constraints |

##### Example MCP tool call

```json
{ "name": "wizard_look_variants", "arguments": { "body": { "gender": "female", "age": 24, "nicheName": "Fitness" } } }
```

##### Poll / completion flow

Synchronous — pass `body.variants[]` to **`wizard_preview_images`**.

##### Credit cost notes

**Free**.

---

#### `wizard_preview_images`

| | |
|---|---|
| **Tool name** | `wizard_preview_images` |
| **REST mapping** | `POST /wizard/niche/preview-images` |
| **Description** | Wizard step 2 — render preview faces for chosen variants. **Free, synchronous.** |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age`, `variants` | — | **Yes** | `variants` from `wizard_look_variants` |
| `nicheId`, `nicheName`, `nicheBio` | string | No | Optional context |

##### Example MCP tool call

```json
{
  "name": "wizard_preview_images",
  "arguments": {
    "body": {
      "gender": "female",
      "age": 24,
      "variants": [{ "label": "Soft", "looks": { "gender": "female", "ethnicity": "Latina" } }]
    }
  }
}
```

##### Poll / completion flow

Synchronous — pick a `referenceUrl` from previews → **`wizard_finalize_poses`**.

##### Credit cost notes

**Free**.

---

#### `wizard_custom_reference`

| | |
|---|---|
| **Tool name** | `wizard_custom_reference` |
| **REST mapping** | `POST /wizard/custom/reference` |
| **Description** | Wizard — custom text reference face. First call free; `regenerate: true` is charged. **Synchronous.** |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age` | — | **Yes** | |
| `referencePrompt` | string | No | Appearance description |
| `regenerate` | boolean | No | `true` triggers paid regeneration |
| appearance fields / `savedAppearance` / `look` | — | No | Structured looks |

##### Example MCP tool call

```json
{
  "name": "wizard_custom_reference",
  "arguments": {
    "body": {
      "gender": "female",
      "age": 26,
      "referencePrompt": "minimalist fashion creator, platinum blonde bob"
    }
  }
}
```

##### Poll / completion flow

Synchronous — use returned `referenceUrl` in **`wizard_finalize_poses`**.

##### Credit cost notes

First call **free**. `regenerate: true` uses `wizardCustomRegenerate` (default **10** credits).

---

#### `wizard_upload_save`

| | |
|---|---|
| **Tool name** | `wizard_upload_save` |
| **REST mapping** | `POST /wizard/upload/save` |
| **Description** | Wizard — create a model from three uploaded photos (auto-normalized to studio references). **Free, async (202).** |

##### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `name`, `photo1Url`, `photo2Url`, `photo3Url` | — | **Yes** | Hosted photo URLs |
| `age`, `gender`, `savedAppearance` | — | No | Optional metadata |

##### Example MCP tool call

```json
{
  "name": "wizard_upload_save",
  "arguments": {
    "body": {
      "name": "UploadClone",
      "photo1Url": "https://cdn.example.com/selfie.jpg",
      "photo2Url": "https://cdn.example.com/portrait.jpg",
      "photo3Url": "https://cdn.example.com/fullbody.jpg"
    }
  }
}
```

##### Poll / completion flow

**Async** — poll **`models_status`** with returned model id.

##### Credit cost notes

**Free**. Upload path models are **not** NSFW-eligible.

---

#### `wizard_finalize_poses`

| | |
|---|---|
| **Tool name** | `wizard_finalize_poses` |
| **REST mapping** | `POST /wizard/finalize-poses` |
| **Description** | Wizard final step — persist model + async 3-pose set. **Free on wizard path.** Same body shape as `models_generate_poses`. |

##### Input schema

Same as **`models_generate_poses`** — `name`, `referenceUrl`, appearance fields, optional pose hints.

##### Example MCP tool call

```json
{
  "name": "wizard_finalize_poses",
  "arguments": {
    "body": {
      "name": "FitCreator",
      "referenceUrl": "https://cdn.modelclone.app/references/prev-1.jpg",
      "gender": "female",
      "age": 24,
      "ethnicity": "Latina",
      "hairColor": "Dark Brown"
    }
  }
}
```

##### Poll / completion flow

**Async** (202) → **`models_status`** until `ready`.

##### Credit cost notes

**Free** on the wizard path (unlike paid `models_generate_poses` on the classic path).

---

### SFW image & video generation

Cross-references: [`11-image-generation.md`](../../public-api/11-image-generation.md), [`12-video-generation.md`](../../public-api/12-video-generation.md), [`13-creator-studio.md`](../../public-api/13-creator-studio.md).

| Tool | REST | Async | Poll |
|------|------|-------|------|
| `generate_image_identity` | `POST /generate/image-identity` | Yes | `wait_for_generation` |
| `generate_recreate` | `POST /generate/recreate` | Yes | `wait_for_generation` |
| `generate_free` | `POST /generate/free` | Yes | `wait_for_generation` |
| `generate_model_caps` | `GET /generate/model-caps` | **Sync** | — |
| `list_styles` | `GET /styles` | **Sync** | — |
| `enhance_prompt` | `POST /generate/enhance-prompt` | **Sync** | — |
| `generate_motion_video` | `POST /generate/motion-video` | Yes | `wait_for_generation` |
| `creator_studio_config` | `GET /generate/creator-studio/config` | **Sync** | — |
| `creator_studio_image` | `POST /generate/creator-studio` | Yes | `wait_for_generation` |
| `creator_studio_enhance` | `POST /generate/creator-studio/enhance` | **Sync** | — |
| `creator_studio_marketplace` | `POST /generate/creator-studio/marketplace` | Yes (1–13 ids) | `wait_for_generation` per id |
| `creator_studio_video` | `POST /generate/creator-studio/video` | Yes | `wait_for_generation` |

**Webhooks:** `integrationCallbackUrl` + optional `integratorWebhookSecret` in `options` (recreate/free/enhance/image-identity) or `body` (motion/creator studio). **Credits:** `get_pricing_generation`.

#### `generate_image_identity`

`POST /generate/image-identity` — legacy target-photo identity recreation. Prefer `generate_recreate` for new work.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Saved model UUID (3 reference photos) |
| `targetImage` | string | yes | Public HTTPS URL of the photo to edit |
| `quantity` | number | no | 1–10 (default 1) |
| `prompt` | string | no | Extra creative guidance |
| `clothesMode` | string | no | Outfit handling (e.g. `reference`) |
| `options` | object | no | Webhook fields and additional REST keys |

Credits: `imageIdentity` (**10**/image). See [`11-image-generation.md`](../../public-api/11-image-generation.md#post-generateimage-identity).

#### `generate_recreate`

`POST /generate/recreate` — core identity recreation.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Saved model UUID (3 reference photos) |
| `sourceImageUrl` | string | yes | Public HTTPS URL of the photo to recreate |
| `count` | number | no | 1–10 (default 1) |
| `outfitMode` | string | no | `model` (default), `source`, or `external` |
| `externalOutfitImageUrl` | string | when `outfitMode: external` | Outfit reference image |
| `extraGuidance` | string | no | Extra creative direction (max 400 chars) |
| `genModel` | string | no | `wan-2.7-image` (default) or `nano-banana-pro` |
| `options` | object | no | Webhook fields and any additional REST body keys |

Credits: `recreateImage` (**10**/image) or `recreateImageNanoBanana` (**16**/image).

```json
{
  "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "sourceImageUrl": "https://cdn.example.com/inspo/pose.jpg",
  "outfitMode": "model",
  "count": 1
}
```

#### `generate_free`

`POST /generate/free` — prompt-driven image with model identity locked by 3 reference photos.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Saved model UUID |
| `prompt` | string | yes | Scene/idea description. Oversize → **400** `PROMPT_TOO_LONG` `{ promptChars, maxPromptChars }` before charge |
| `genModel` | string | no | `nano-banana-pro` (default), `wan-2.7-image`, `seedream-4.5-edit` |
| `refImageUrls` | string[] | no | When provided (non-empty), **replaces** the model's 3 identity photos. MCP sets `replaceIdentityRefs: true` on the REST body. Dashboard / HTTP default remains append |
| `aspectRatio` / `resolution` | string | no | Engine-dependent — see [`11-image-generation.md`](../../public-api/11-image-generation.md). WAN 2.7 image honors `aspectRatio` (pads refs + forwards `aspect_ratio`) |
| `enhance` | boolean | no | Default `true` (+`enhancePromptDefault` **1** credit once per request) |
| `styleId` | string | no | Style preset id from `list_styles` — prepended before the enhancer when `enhance` is on, appended to the final prompt when off. Unknown id → **400** `STYLE_NOT_FOUND` |
| `styleStrength` | number | no | 0–1, default **0.7**; below **0.4** the style is a subtle hint |
| `count` | number | no | 1–8 (default 1) |
| `options` | object | no | Webhook fields and additional REST keys |

Prompt ceilings for this tool: `nano-banana-pro` **10000**, `wan-2.7-image` **5000**, `seedream-4.5-edit` **1800**. Call `generate_model_caps` for live `promptMaxChars` / ratios / ref limits. Do not send more than 3 custom refs if you meant to replace identity — they replace, they are not appended.

#### `generate_model_caps`

`GET /generate/model-caps` — free, synchronous. Returns the `/generate/free` capability map: `maxRefs`, `ratios`, `resolutions`, `supportsRatio`, `supportsResolution`, and `promptMaxChars` per engine. WAN 2.7 image reports `supportsRatio: true`. Call this before submitting `generate_free`. No parameters.

#### `list_styles`

`GET /styles` — free, synchronous. Returns the curated style preset catalog (`id`, `name`, `description`, `category` — smartphone / editorial / cinematic / street / documentary / analog / studio). Pass an id as `styleId` on `generate_free`, `creator_studio_image`, or `enhance_prompt` with optional `styleStrength` 0–1 (default **0.7**; below **0.4** the style is applied as a subtle hint). Unknown id → **400** `STYLE_NOT_FOUND` with `validStyleIds`. No parameters.

#### `enhance_prompt`

Sync — returns `body.enhancedPrompt`. `options`: `mode`, `genModel`, `modelLooks.gender`. Credits: `enhancePromptDefault` (**1**).

#### `generate_motion_video`

`POST /generate/motion-video` — SFW motion-control video.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `imageUrl` | string | yes | Still image to animate |
| `videoUrl` | string | yes | Driving reference clip (public HTTPS) |
| `prompt` | string | no | Motion description |
| `duration` | number | no | Output seconds 2–15 (default 5) |
| `skipSeconds` | number | no | Skip from start of driving clip |
| `seed` | number | no | Fixed seed |
| `options` | object | no | Webhook fields, trim fields, etc. |

Credits: `motionXPerSec` (**9.5**/s) × duration. Poll `wait_for_generation` on `body.generationId`.

#### `creator_studio_config`

No parameters. Returns the live image engine ids, aspect/resolution controls, reference limits, input requirements, current credit tiers, creative modes, marketplace scope counts, and a `video` family catalog (durations, modes, prompt ceilings). Veo 3.1 durations are **4 / 6 / 8** only (`5` is invalid). Call this before choosing or claiming support for an engine. Free and synchronous.

#### `creator_studio_image`

Typed parameters cover the common image path and model-aware enhancer:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | yes* | Raw user intent; required unless supplied in legacy `body` |
| `generationModel` | enum | no | One of the ten Creator Studio image models |
| `aspectRatio`, `resolution` | string | no | Model-specific output settings |
| `referencePhotos` | string[] | no | Up to 8 hosted reference images |
| `inputImageUrl`, `maskUrl` | string | no | Edit/remix inputs |
| `numImages` | integer | no | 1–4 |
| `enhancePrompt` | boolean | no | Run the selected model's server-side enhancer |
| `styleId` | string | no | Style preset id from `list_styles` — feeds the enhancer when `enhancePrompt` is on; appended to the final prompt otherwise |
| `styleStrength` | number | no | 0–1, default **0.7**; below **0.4** the style is a subtle hint |
| `mode` | enum | no | Creative mode such as `product_shot`, `moodboard_pin`, or `hero_banner` |
| `scope` | enum | no | `main`, `product-images`, `aplus`, or `full-set` |
| `asset`, `productContext`, `brandContext` | string | no | Structured creative context for enhancement |
| `body` | object | no | Legacy/additional model-specific fields; named parameters override matching keys |

Full model/resolution matrix: [`13-creator-studio.md`](../../public-api/13-creator-studio.md).

#### `creator_studio_enhance`

Synchronous, enhancer-only preview. Required `prompt`; optional typed `generationModel`, `mode`, `scope`, `asset`, `productContext`, `brandContext`, references, and `batchIndex` / `batchTotal`. Returns `enhancedPrompt` and charges only `enhancePromptDefault`; an unavailable enhancer returns the original prompt with `fallback: true` and a refund.

#### `creator_studio_marketplace`

Submits a coordinated marketplace set from one shared brief:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | yes | Short product/campaign intent |
| `scope` | enum | no | `main` (1), `product-images` (6), `aplus` (8), `full-set` (13; default) |
| `referencePhotos` | string[] | no | Up to 8 hosted product images |
| `inputImageUrl` | string | no | Primary hosted product image |
| `productContext`, `brandContext` | string | no | Shared product and brand locks |
| webhook fields | string | no | Applied to every generated asset |

The result includes labeled `generations[]`; call `wait_for_generation` for each id. Cost is `creatorStudioGptImage2 × count + enhancePromptDefault` (one enhancement per set).

#### `creator_studio_video`

Video continues to accept the full REST `body`; its rate bucket is `cinematic`. Use `creator_studio_extend`, `creator_studio_4k`, and `creator_studio_1080p` for post-processing. Read `creator_studio_config` → `video` for the live family catalog.

- **Veo 3.1** (`family: "veo31"`): `durationSeconds` **4, 6, or 8** only (default 8). **`5` is rejected** before charge (`Veo 3.1 duration must be 4, 6, or 8 seconds.`). `ref2v` and `extend` stay 8s. Do not send 5.
- **Video-X** (`family: "videox"`): allowed here and on Reel Recreate (`generate_video_recreate`) and Marketing Studio (`engine: "videox"`). Modes `t2v` \| `i2v` \| `fl2v` \| `r2v` \| `reel-recreate`, duration **5 / 10 / 15**. Video-X is not available on NSFW tools.
- **Prompt ceilings** (400 `PROMPT_TOO_LONG` `{ promptChars, maxPromptChars }` before charge): Kling 3.0 **2500**, Veo 3.1 **10000**, Seedance 2.5 **30000**. Do not silently trim the user prompt below these values.

#### `generate_video_recreate`

`POST /generate/video-recreate` — looks photo + Gemini Analyze JSON prompt. Default family Seedance 2.5; `family: "videox"` snaps duration to 5/10/15. Inspiration reel is not sent to the engine.

---

### Agent ergonomics (submit hints, cost preflight, recovery)

Three conventions apply across the toolset:

1. **Self-describing submits.** Every generation-submit tool response includes `agent: { pollWith, generationIds, note? }` — call the named poll tool with each id (batch submits return one pollable child id per output; the parent set id is not pollable). No need to search for the right poller.
2. **Cost preflight.** `estimate_cost` (free) quotes the exact credits a submit would charge before you spend: `{ kind: "creator-studio-image" | "creator-studio-video" | "creator-studio-marketplace" | "marketing-video" | "marketing-image", params: { …same fields as the generate tool } }` → `{ credits, approximate, breakdown, note }`. Quote batch jobs to the user before submitting.
3. **Structured recovery.** Failed responses include a `recovery` field with the concrete next action (e.g. insufficient credits → check `get_me`, invalid enum → fetch `creator_studio_config` / `marketing_studio_config`, NSFW age gate → `confirm_adult`, 429 → wait and retry once). Follow it instead of blind retries.

REST: `POST /pricing/estimate` (free). CLI: `modelclone estimate --kind … --params '{…}'`.

---

### Uploads — typed tools

REST reference: [Uploads](../../public-api/06-uploads.md). These are the bridge between files that exist only in the agent/chat context (or on remote hosts) and the hosted URLs every generation tool expects.

#### `upload_media`

`POST /upload/base64` — upload from raw base64 (`base64Data` + optional `fileName`/`contentType`) or a full `dataUrl`. Returns `{ url }` usable in any `imageUrl` / `referencePhotos` / product-image field. **This is the tool to use when the user attached an image to the conversation** — read/encode it and upload; never tell the user ModelClone "cannot use attached files". Default cap ~3.5 MB binary (JSON body limit).

#### `upload_from_url`

`POST /upload/from-url` — the server fetches a public https file URL and mirrors it into ModelClone storage. Use for website/CDN assets the user links to, or when a source host is not reachable by generation providers. Private hosts and non-https URLs are rejected.

Decision guide: chat attachment → `upload_media` · remote link → `upload_from_url` · local file with CLI access → `modelclone upload <file>`.

---

### Marketing Studio — typed tools

REST reference: [Marketing Studio](../../public-api/26-marketing-studio.md). Branded ad video/image with reusable products, presenter avatars, and curated hook/setting setup items (Higgsfield Marketing Studio parity).

#### Typical sequence

1. `marketing_products_fetch` (URL import; poll `marketing_products_get` until `status: "ready"`) or `marketing_products_create`.
2. Optionally `marketing_avatars_list` (pick a preset) or `marketing_avatars_create` (custom presenter from portraits).
3. Optionally `marketing_hooks_list` / `marketing_settings_list` — valid only for `ugc`, `ugc_how_to`, `ugc_unboxing`, `product_review`, `ugc_virtual_try_on`.
4. `marketing_studio_video` (or `marketing_studio_image`) → poll with `wait_for_generation`.

#### `marketing_studio_config`

Free discovery: mode list (with per-mode `allowsSetupItems`), duration/aspect/resolution limits, and live credit rates.

#### `marketing_products_list` / `marketing_products_get` / `marketing_products_create` / `marketing_products_fetch`

`marketing_products_fetch` takes a public https `url`, extracts title/description/images (OG + JSON-LD, Grok fallback), mirrors images to ModelClone storage, and dedupes by URL — repeat fetches return the existing product. Import runs in the background: the returned product starts as `pending`; poll `marketing_products_get` until `ready` (or `failed` with `errorMessage`).

#### `marketing_avatars_list` / `marketing_avatars_create`

Presets are global (`type: "preset"`); customs belong to the caller. `marketing_avatars_create` takes `name` + up to 4 hosted portrait `imageUrls` (first = primary).

#### `marketing_hooks_list` / `marketing_settings_list`

Read-only curated catalogs; optional `search` filter. Hook text is prepended to the prompt at generation time — it never replaces the prompt.

#### `marketing_studio_video`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | no* | Creative brief; optional when `productIds` provide context |
| `mode` | enum | no | 9 slugs, default `ugc` |
| `engine` | enum | no | `seedance` (default), `geminiOmni`, or `videox` (Video-X) — limits/credits from `marketing_studio_config` → `video.engines` |
| `productIds` | string[] | no | Up to 3 ready products |
| `avatars` | array | no | Max 1: `{ id, type: "preset" \| "custom" }` |
| `hookId`, `settingId` | string | no | UGC-family modes only; `400` otherwise |
| `durationSeconds` | integer | no | Seedance 4–30 (default 8); Gemini Omni `4\|6\|8\|10`; Video-X `5\|10\|15` |
| `aspectRatio` | string | no | Seedance: `auto`, `21:9`, `16:9`, `4:3`, `1:1`, `3:4`, `9:16`. Omni: `16:9`\|`9:16`. Video-X: `16:9`\|`9:16`\|`1:1`\|`4:3`\|`3:4`\|`21:9`. Default `9:16` |
| `resolution` | string | no | Seedance: `480p`\|`720p`. Omni: `720p`\|`1080p`\|`4k`. Ignored for Video-X. Default `720p` |
| `generateAudio` | boolean | no | Seedance only (default `true`). Ignored for `geminiOmni` and `videox` (always native audio) |
| `videoxQuality` | enum | no | Video-X only: `turbo` (default) or `quality` |
| `enhancePrompt` | boolean | no | Default `true` — Grok marketing-video enhancer |

For other languages after a finished clip, use `marketing_studio_video_translate`. The former `voiceover` param was removed and now returns `400`.

Cost: Seedance `marketingStudioVideo{480p\|720p}PerSec × duration`; Omni uses `geminiOmni{720p\|1080p\|4k}PerSec`. Returns `generation.id`; poll `wait_for_generation`.

#### `marketing_studio_video_translate`

`POST /generate/marketing-studio/video/translate` — HeyGen lipsync dubbing of a completed Marketing video. **Pass-through cost only** (no margin).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `generationId` | string | yes | Completed `marketing-studio-video` (or prior translate) id |
| `language` | string | yes | Target language display name from config `translation.languages` (e.g. `"Spanish"`) |
| `mode` | enum | no | `speed` (~5 cr/s) or `precision` (default, ~10 cr/s) |

Returns `generation.id` (`type: marketing-studio-video-translate`); poll `wait_for_generation`. Quote cost first with `estimate_cost` kind `marketing-video-translate`.

#### `marketing_studio_image`

GPT Image 2 ad image with product references. `hookId`/`settingId` are rejected (video-only). Flat `marketingStudioImage` credits.

---

### Creative skills (`get_creative_skill`)

Read-only. No REST call and no credit spend. Call it before the matching agent step.

| `skill` | When | Topics |
|---|---|---|
| `copy` | Before `marketing_agent_copy` | none |
| `direction` | Before `marketing_agent_plan` | none |
| `studio` | Before `marketing_agent_create` | `entities`, `interview-flow`, `ugc-realism`, `unsupported` |
| `brand`, `logo`, `social` | Brand, logo, or social deliverables | the catalog topic only |

The server still applies the copy and direction skills when it drafts or plans. This tool lets the MCP client read the same text.

---

### Marketing Studio — conversational agent (`marketing_agent_*`)

REST reference: [Marketing Studio — conversational production sessions](../../public-api/26-marketing-studio.md#conversational-production-sessions). These tools proxy authenticated `/api/v1/marketing-studio/agent*` routes 1:1 (same gates, credits, and revision rules as the Marketing Studio app).

**Reasoning model:** agent copy/plan/quality/repair/finish reasoning uses OpenRouter `anthropic/claude-opus-5.5` (`MARKETING_AGENT_MODEL`). Each reasoning call reserves **120 credits** up front and settles to actual helper pricing from provider usage.

#### Review modes

| Mode | Behavior |
|------|----------|
| **`manual`** | Stops at an **approval** gate before motion. The user must call `marketing_agent_review` with `decision: "approve"` and the exact current `assetIds` (audio + frame IDs). A `continue` / `next` message **never** records approval. |
| **`autonomous`** | The server runs multimodal **quality** inspection on actual media. Passed/failed checks are appended to `session.reviews` with `reviewer: "ai"` and `model: "anthropic/claude-opus-5.5"`. Before motion, `approveAgent` records production approval with `reviewer: "ai"` — **not** a user approval. Bounded by `budgetCredits` and per-step `maxAttempts` (1–10). |

Create requires `reviewMode`, positive `budgetCredits`, and `maxAttempts`. Optional locked `script`, `voice` (`voiceId`, optional `modelId`, optional `ttsModel: "eleven_v4"`), `endCardGenerationId`, `productionId`, and `sourceRequest` (products, avatars, brand, speech language, aspect ratio, etc.). A presenter voice is `voiceId` `mkt_<avatarId>`; `modelId` may be omitted until speech is recorded.

#### Messaging vs advancing

- **`marketing_agent_message`** — Creative feedback, session limit changes (`settings.maxAttempts`, `settings.budgetCredits`), and owned settings (`settings.voice`, `settings.brandId`, `settings.endCard`, including `settings.endCard.productId` for the ending photograph, …). If the entire message matches `continue`, `next`, `proceed`, `go ahead`, `start`, or `run the next step` (case-insensitive), the server runs the **next executable** step with `execute: true`. Any other text is stored as feedback and handled as a **feedback** step (conversational reply; may revise a saved plan when explicitly requested), except while a spoken line is missing, a manual model choice is pending, the ending is waiting, a charge is unresolved, or the higher-tier step is waiting, or a blocked step is still closed, or a step is still running, including a clip, reference, frame, join, card, lip-sync, or cut still with the provider, or a prompt repair is waiting, or an inspection is the next step, or the final cut is the next step. A note during that repair is kept for the rewrite. A note before an inspection is kept for that check. A note before the final cut is kept for the edit ranges. That note drops a saved cut and does not start one. A note while a lip-sync quote is next, or while that quote is waiting for approval, stays in the transcript. It does not approve the quote, call the model, or start the sync. A reply before a production exists ignores an invented plan and does not block the session. A reply that includes a plan this ad cannot use leaves the saved plan and does not block the session. A reply that changes a locked spoken line leaves that line and does not apply the plan. A chat reply that changes the scene after product photos or starting frames exist leaves those images in place. A chat reply after the video has been rendered leaves that video in place and does not block the session. Those notes stay in the transcript and do not start feedback or a generation. Before a production exists, `settings.budgetCredits` follows the spend rule: autonomous mode switches the engine, and manual mode keeps it until the message is `use Seedance 2.5` or `use Video X`. After a production exists, the engine stays. Before a production exists, `settings.durationSeconds` sets the shot length. Video X accepts 5, 10 or 15. Seedance accepts a whole number from 4 to 30. That length stays locked once a production exists. A chat reply that changes that length leaves it and does not apply the plan. Before a production exists, `settings.aspectRatio` sets the frame shape to 9:16, 16:9, or 1:1. The final cut uses that shape. It stays locked once a production exists. A shape the join cannot deliver is stored as 9:16. The ending card and the join use that same shape. A chat note before that manual model choice, while the ending is waiting for a product and brand, or while a charge is still unresolved, stays in the transcript and does not start a feedback generation. An unresolved charge stays closed. When `nextAction.step` is `audio` and the shot has no spoken line, `settings.speechLine` (1–2000 characters) stores that line. It does not start synthesis or replace a locked line.
- **`marketing_agent_advance`** / stage tools — Pass `revision` (optimistic concurrency). **`execute: true`** is required to spend credits or call providers; omitting it or `execute: false` returns current state only. Stage tools are thin wrappers around the same advance orchestrator with a fixed step path. That step runs only when it is the current next action. `brief` runs only for copy. `captions` and `export` run only for finish. Finish and sync cannot run early.
- **`marketing_agent_advance`** with **`run: true` / `run: false`** (no `step`) — toggles autonomous background runner (`202`); only valid in **`autonomous`** mode. Stops after the in-flight accepted step.

Manual sync spend requires `marketing_agent_review` when `nextAction.scope === "sync"` (quote hash in `assetIds`). Rejecting the current inputs or that quote leaves the approval open. It does not block the session or start motion or sync.

#### Typical `nextAction.step` order

The server computes the next safe step from session state and linked production (`session.nextAction`). Do not skip ahead — out-of-order execution returns `MARKETING_AGENT_ORDER`.

**Pre-production (no `productionId` yet):**

1. **`copy`** — draft or lock spoken script (`marketing_agent_copy`; `marketing_agent_brief` hits the same handler via step alias).
2. **`plan`** — shared scene + per-segment action outlines; creates the production record.

**Production (after plan is saved):**

3. **`audio`** — Eleven v4 speech per segment; inspect audible words and duration before motion dependencies. If the shot has no spoken line, `canExecute` is false until `settings.speechLine` stores one. That save does not record speech. A normal chat message is not stored as that line and does not start a feedback generation. A missing line does not use an attempt.
4. **`components`** → **`quality`** — for each handled prop: generate reference, then inspect actual image evidence. A reference generation that failed is prepared again. One that is still running waits.
5. **`frames`** (or **`continuation`** then **`frames`** for later segments) → **`quality`** — starting frames from approved components or the real prior ending frame. A failed first frame is prepared again. A failed later frame extracts the previous ending frame again. A failed extraction can be tried again. An extraction that is still running waits. A frame that is still running waits.
6. **`approval`** (manual only) or autonomous **`quality`** + internal `approveAgent` — exact audio + references before motion.
7. **`render`** — submit motion for the current approved segment (recover pending jobs instead of duplicating). A generation that failed is rendered again from those inputs. A generation that is still running stays waiting.
8. **`upgrade`** — only after a passed Seedance 720p draft whose delivery tier is 1080p. A one-video render uses that finished video. Separate clips offer the shot that just passed, before the next shot starts. The next shot waits until that 1080p clip finishes and passes review. A repair of a shot that already moved to 1080p renders again at 1080p. Manual mode must pass `upgrade` `accept` or `decline` on `marketing_agent_advance` (or the chat phrases `regenerate` / `keep the draft`). Continue does not spend in manual mode. In autonomous mode, continue runs that higher tier. Video X does not enter this step.
9. Repeat **continuation → frames → quality → render** for multi-segment `separate_clips` plans.
10. **`assembly`** — stitch accepted segments in order. A failed join can start again. A join that fails before a file exists is saved as failed and can start again. A join that is still running stays closed. A 720p draft stays 720p. A shot rendered at 1080p makes the join 1080p. Final editing uses that same canvas, including after lip-sync. The lip-sync record keeps that canvas.
11. **`sync`** — when speech language is not English (or lip-sync is otherwise required); manual mode may require quote approval via **`review`**. A failed lip-sync prepares a new quote. One that is still running waits.
12. **`endcard`** — branded ending from one owned product and brand logo. A failed card is drawn again. A card that is still drawing waits. If either is missing, the step asks and does not draw a card or use an attempt. A locked spoken line longer than 80 characters asks for a 1–80 character headline and is not shortened. The shot line counts when the session has no script. A line that already fits is the headline. A slogan, call to action, or product line is drawn only when the user saved it. The model is not called to write the card. An omitted call to action stays empty. `settings.endCard.productId` can name the product after the video exists.
13. **`finish`** — final edit ranges, captions, and export (`marketing_agent_finish`, `marketing_agent_captions`, and `marketing_agent_export` route to this step; `captions` / `export` are step aliases). A cut the worker marked failed starts again from the saved ranges. A cut that is still running waits.

**`quality`** can also run whenever completed media exists that lacks a passing review for its current URL (interleaved after generation). A voice take that is silent or does not match the locked line is detached before motion, and the next continue records that same line again. A take already used in a clip stays attached. Saving the spoken line, detaching a missed take, or attaching the new recording changes only that audio. Approved product photos and starting frames stay. A failed product or starting-frame reference, before motion, schedules a prompt repair in both review modes. The next continue rewrites that prompt and does not inspect the same image again. A failed clip does the same for its motion prompt in both review modes. Manual review does not render on that continue. A failed ending card stays in place until the user saves a different headline or card text. That save removes only the failed card. A passed card, and a card already used in final editing, stay locked. A failed final cut is removed when the ad is ready to be cut again. The next continue measures the source again and does not reuse that file. A failed lip-sync is removed the same way. The next continue prepares a new quote and does not reuse that file. **`repair`** runs after a recorded failed review of that exact asset and does not waive the check.

#### Session control tools

| Tool | REST | Purpose |
|------|------|---------|
| `marketing_agent_create` | `POST /marketing-studio/agent` | Start session (`brief`, `reviewMode`, `budgetCredits`, `maxAttempts`, …) |
| `marketing_agent_get` | `GET /marketing-studio/agent/:sessionId` | Resume; read `nextAction`, `reviews`, `receipts`, `blocker`, `production` |
| `marketing_agent_message` | `POST …/:sessionId/message` | Feedback, settings, or whole-message **continue** |
| `marketing_agent_advance` | `POST …/:sessionId/advance` | Generic advance; optional `run`, `retry`, `recover`, `step`, `upgrade` `accept` or `decline` |
| `marketing_agent_review` | `POST …/:sessionId/review` | Manual **`approve`** / **`reject`** with exact `assetIds` |
| `marketing_agent_finish` | `POST …/:sessionId/finish` | Advance **`finish`** when `nextAction` allows |

Shared body fields on stage tools: `{ sessionId, revision?, execute? }` (plus advance-only flags on `marketing_agent_advance`).

#### Stage tools (fixed REST step paths)

Each row is `POST /marketing-studio/agent/:sessionId/steps/<step>` with the same advance body.

| Tool | Step path | Executes when `nextAction.step` is |
|------|-----------|-------------------------------------|
| `marketing_agent_brief` | `steps/brief` | aliases → **`copy`** |
| `marketing_agent_copy` | `steps/copy` | **`copy`** |
| `marketing_agent_plan` | `steps/plan` | **`plan`** |
| `marketing_agent_audio` | `steps/audio` | **`audio`** |
| `marketing_agent_components` | `steps/components` | **`components`** |
| `marketing_agent_frames` | `steps/frames` | **`frames`** |
| `marketing_agent_quality` | `steps/quality` | **`quality`** |
| `marketing_agent_render` | `steps/render` | **`render`** |
| `marketing_agent_continuation` | `steps/continuation` | **`continuation`** |
| `marketing_agent_assembly` | `steps/assembly` | **`assembly`** |
| `marketing_agent_sync` | `steps/sync` | **`sync`** |
| `marketing_agent_captions` | `steps/captions` | aliases → **`finish`** |
| `marketing_agent_endcard` | `steps/endcard` | **`endcard`** |
| `marketing_agent_export` | `steps/export` | aliases → **`finish`** |

CLI parity: `modelclone marketing agent <operation> [sessionId] --file body.json` (see `cli/src/commands/marketing.js`).

#### Operator checklist

1. `marketing_studio_config` + product/avatar/hook lists — build `sourceRequest`.
2. `marketing_agent_create` → save `session.id` and `revision`.
3. Loop: `marketing_agent_get` → read `nextAction` → call the matching stage tool or `advance` with `{ revision, execute: true }` **only** when `canExecute` is true.
4. Manual mode: on `requiresReview`, call `marketing_agent_review` (never infer approval from chat text).
5. On `blocker` or `MARKETING_AGENT_RETRY_LIMIT`, inspect evidence; adjust inputs or authorized `settings.maxAttempts` / `settings.budgetCredits` via `message` before retrying.

Related lower-level workflow (no conversational session): **`marketing_production_*`** tools and resource `modelclone://v1/marketing-preproduction`.

---

### NSFW access (all NSFW tools)

Same gate as REST ([NSFW Studio](../../public-api/14-nsfw.md#access-requirements)):

| Layer | Requirement | Typical `403` `code` |
|-------|-------------|------------------------|
| Account | First purchase + 18+ confirmation | `NSFW_NEEDS_PURCHASE`, `NSFW_NEEDS_AGE_CONFIRMATION` |
| Classic LoRA pipeline (`nsfw_generate`, `nsfw_train_lora`) | AI-generated model, 18+ persona, trained LoRA (`nsfwUnlocked`) | — (message: train LoRA first) |
| v2 stills + NSFW video sessions | NSFW-verified model + all three NSFW reference photos | `NSFW_NOT_VERIFIED`, `NSFW_REFS_INCOMPLETE` |

Age confirmation is a one-time dashboard action — not grantable via API key.

**Webhooks:** any generation-creating POST body (including `nsfw_video_create_session` and `nsfw_video_session_action` when it creates a generation) may include `integrationCallbackUrl` and `integratorWebhookSecret`. See [Webhooks](../../public-api/05-webhooks.md).

**Sanitization (API-key / MCP responses):** provider fields are stripped globally. NSFW Video pipeline generation rows additionally return `prompt: null` and omit `engine` — the composition is proprietary. Session objects are not sanitized beyond normal provider stripping. Details in [NSFW Video](../../public-api/15-nsfw-video.md#notes).

---

### NSFW studio — typed tools

REST reference: [NSFW Studio](../../public-api/14-nsfw.md). Costs: `get_pricing_generation`.

#### `nsfw_generate`

`POST /nsfw/generate` — classic NSFW image with a trained LoRA.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Model UUID |
| `prompt` | string | yes | Scene prompt — include the model's LoRA trigger word |
| `quantity` | number | no | `1` (default) or `2` |
| `attributes` | string | no | Comma-separated appearance/scene chips |
| `sceneDescription` | string | no | Short scene note |
| `options` | object | no | `loraStrength`, `quickFlow`, `resolution`, `postProcessing`, webhook fields |

**Response:** `{ success, generation, generations[], creditsUsed, creditsRemaining, imageQuantity }` — poll `wait_for_generation` on each `generation.id`.

**Completion:** `wait_for_generation(generationId)` — typically 1–3 minutes.

#### `nsfw_generate_video`

`POST /nsfw/generate-video` — animate a still image into a short NSFW video (5 s or 8 s).

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | NSFW-eligible model |
| `imageUrl` | string | yes | Source image URL (typically a completed generation `outputUrl`) |
| `prompt` | string | no | Motion description; natural-motion default if omitted |
| `duration` | number | no | `5` (default) or `8` |
| `integrationCallbackUrl` / `integratorWebhookSecret` | string | no | Webhook fields |

**Response:** `{ success, generationId, creditsUsed, creditsRemaining, duration }` → `wait_for_generation(generationId)`.

#### `nsfw_train_lora`

`POST /nsfw/train-lora` — start LoRA identity training (long-running).

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | AI-generated model |
| `loraId` | string | no | Target LoRA row; omit for legacy per-model path |

**Response:** `{ success, deferred: true, triggerWord, creditsUsed, message }` (HTTP 202 semantics).

**Poll:** `nsfw_training_status` every ~60 s until `status: "ready"` or `failed`. Training prep/training takes ~1–6 h by tier.

> LoRA setup (create LoRA, upload/generate training images, set active) is **not** exposed as typed MCP tools — use `api_v1_request` with paths from `modelclone://v1/route-catalog` and shapes from `modelclone://v1/openapi`. See [LoRA training](../../public-api/14-nsfw.md#lora-training).

#### `nsfw_training_status`

`GET /nsfw/training-status/:modelId`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Model UUID |
| `loraId` | string | no | Specific LoRA row id (REST query `?loraId=`) |

**Response:** `{ success, status, loraUrl?, triggerWord?, nsfwUnlocked, loraId?, firstLoraBonus?, preprocessing? }` where `status` is `none` \| `awaiting_images` \| `images_ready` \| `training` \| `ready` \| `failed`.

#### `nsfw_v2_preset`

`POST /nsfw-v2/presets` — fast preset still from three NSFW reference photos (no LoRA).

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | NSFW-verified model with complete refs |
| `presetId` | string | yes | Preset from v2 catalog (222 presets — category prefixes in REST doc) |
| `aspectRatio` | string | no | `1:1`, `4:5`, `9:16` (default), `16:9`, `3:4`, `2:3` |
| `count` | number | no | 1–8 images (default 1); 6 credits each |
| `integrationCallbackUrl` / `integratorWebhookSecret` | string | no | Webhook fields |

**Response:** `{ success, generationIds[], generations[], creditsCost, prompt, preset: { id, name, categoryId } }` — poll each id with `wait_for_generation`.

#### `nsfw_v2_undress`

`POST /nsfw-v2/undress` — nude version of an existing image. 15 credits per image.

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | |
| `sourceImageUrl` | string | yes | Public https URL of source image |
| `count` | number | no | 1–8 (default 1) |
| webhook fields | string | no | As above |

**Response:** `{ success, generationIds[], generations[], creditsCost }`.

#### `nsfw_v2_free_prompt`

`POST /nsfw-v2/free-prompt` — free-text still with optional AI prompt enhancement. 6 credits per image.

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | |
| `prompt` | string | yes | Your scene description |
| `aspectRatio` | string | no | Same values as presets |
| `count` | number | no | 1–8 (default 1) |
| `enhance` | boolean | no | `true` (default) = AI rewrite before render; `false` = light wrap only |
| webhook fields | string | no | As above |

**Response:** `{ success, generationIds[], generations[], creditsCost, prompt }` — when `enhance: true`, the echoed `prompt` is pre-enhancement; final text is on the generation record (visible in SPA; integrator generation rows follow normal sanitization rules).

---

### NSFW video preset sessions — typed tools

REST reference: [NSFW Video Sessions](../../public-api/15-nsfw-video.md). Multi-step flow: **preview → select → (optional edit) → approve → submit**.

#### Session status machine

```
previewing → editing → approved → submitted → completed
     │           │                      │
     └───────────┴──────────────────────┴──→ failed
```

Poll `nsfw_video_get_session` every **3–5 s** during previews/edits; every **5–10 s** after submit.

#### Preset catalog — `nsfw_video_presets`

`GET /nsfw-video/presets` — no parameters.

**Response:** `{ success, presets: [{ id, key, label, thumbnailUrl, durationSeconds }] }`

Use the returned **`id`** (e.g. `cpre_…`) as `presetId` when creating a session — not the stable `key`. Public scenario slugs (availability depends on admin activation):

| `key` | `label` |
|-------|---------|
| `frontal-dildo-riding` | Frontal Riding POV |
| `frontal-dildo-riding-static` | Frontal Dildo Ride Static |
| `bareback-dildo-riding` | Bareback Dildo Riding |
| `anal-dildo-riding` | Anal Dildo Riding |
| `blowjob` | Sucking Cock POV |
| `fingering` | Fingering |

Only presets with `isActive` and a live reference video appear in the list.

#### `nsfw_video_create_session`

`POST /nsfw-video/sessions` — extracts first frame, starts **free** batch of 3 preview frames.

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | NSFW-verified + complete refs |
| `mode` | string | yes | `preset` or `recreate` |
| `presetId` | string | preset mode | `id` from `nsfw_video_presets` |
| `uploadedVideoUrl` | string | recreate mode | Public https source clip |
| `uploadedVideoDurationSeconds` | number | recreate mode | Clip length, max **15 s** |
| `audioEnabled` | boolean | no | Recreate only; default `true`. Presets always use curated audio |
| webhook fields | string | no | Apply to the preview generation |

**Response:** `{ success, session }` with `status: "previewing"`, empty `previewImageUrls` until batch completes.

#### `nsfw_video_get_session`

`GET /nsfw-video/sessions/:id`

| Parameter | Type | Required |
|-----------|------|----------|
| `sessionId` | string | yes |

**Response:** `{ success, session }` — reconciles in-flight preview/edit/final work. Key fields: `status`, `previewImageUrls[]`, `selectedPreviewUrl`, `currentFrameUrl`, `editHistory[]`, `finalGenerationId`, `sourceVideoDurationSeconds`, `creditsSpentOnPreviews`, `creditsSpentOnEdits`.

When `status === "completed"`, fetch video URL via `get_generation` / `wait_for_generation` using `finalGenerationId`.

#### `nsfw_video_session_action`

`POST /nsfw-video/sessions/:id/<action>`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | yes | Session UUID |
| `action` | enum | yes | See table below |
| `body` | object | no | Action-specific fields |

| `action` | Body | Credits | Effect |
|----------|------|---------|--------|
| `select-preview` | `{ previewUrl }` — must be in `previewImageUrls` | free | → `editing`; sets `currentFrameUrl` |
| `regenerate-previews` | `{}` | 20 | Fresh batch of 3; → `previewing` |
| `edit-frame` | `{ prompt, refImageUrl? }` | 10 | Async edit; poll session until `currentFrameUrl` updates |
| `approve` | `{}` | free | → `approved` (requires working frame, no pending edit) |
| `submit` | `{}` | `ceil(duration × nsfwVideoPerSec)` (default 78.75) | Final 720p video; → `submitted`; sets `finalGenerationId` |

**Typical MCP sequence:**

1. `nsfw_video_presets` → pick `presetId`
2. `nsfw_video_create_session` → loop `nsfw_video_get_session` until `previewImageUrls.length === 3`
3. `nsfw_video_session_action` `{ action: "select-preview", body: { previewUrl } }`
4. (optional) `edit-frame` → poll until edit completes
5. `nsfw_video_session_action` `{ action: "approve" }`
6. `nsfw_video_session_action` `{ action: "submit" }` → poll until `status === "completed"`
7. `wait_for_generation(finalGenerationId)` → `outputUrl`

**Sanitization:** generations created by this pipeline (`nsfw-video-preview`, `nsfw-video-frame-edit`, `nsfw-video-preset`, `nsfw-video-recreate`) return `prompt: null` and omit internal `engine` in API-key responses. User-supplied edit prompts in `editHistory[].promptUsed` are returned on the session object.

---

### NSFW routes without typed MCP tools

Use `api_v1_request` + `modelclone://v1/openapi` for: LoRA CRUD, training-image upload/generation, nudes pack, prompt planner, advanced NSFW image, extend-video, motion-video, and NSFW video recreate when you prefer raw REST over the session action enum. Full route list: `modelclone://v1/route-catalog`.

---

### ModelClone-X (`mcx_*`)

Character-identity image generation with optional trained character LoRA. See **`docs/public-api/16-modelclone-x.md`** for full REST field reference.

#### Typical sequence

1. `mcx_config` — pricing, limits, feature flags (call once per session).
2. `mcx_generate` with a `body` object — returns `generationIds` (async).
3. Poll each id with `mcx_status` **or** `wait_for_generation` / `get_generation`.

#### `mcx_config`

No parameters. `GET /modelclone-x/config` — live credit pricing, step/CFG limits, training image counts, and whether image-to-prompt is enabled.

#### `mcx_generate`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Common fields: `prompt`, `aspectRatio`, `qty` (1–4), `modelId`, `characterLoraId`, `preOptimized`, `useCustomPrompt`, `modelcloneXImg2Img`, `inputImageUrl`. See OpenAPI resource for the full schema. |

`POST /modelclone-x/generate`. Returns generation ids immediately. Character mode requires a trained LoRA (`modelId` + `characterLoraId`). Pass `integrationCallbackUrl` in `body` for webhook completion instead of polling.

#### `mcx_status`

| Parameter | Type | Description |
|-----------|------|-------------|
| `generationId` | string | Required — from `mcx_generate` response |

`GET /modelclone-x/status/:generationId`. MCX-specific status view; `get_generation` works as a cross-type poll target too.

---

### img2img (`img2img_*`)

Outfit/scene transformation: re-render a source photo with your character's identity while preserving outfit, pose, and scene. Two-step flow (describe → generate) or single-step generate. See **`docs/public-api/17-img2img.md`**.

**Prerequisites:** trained character LoRA (`triggerWord`, and optionally `modelId` with NSFW LoRA unlocked).

#### Typical sequence (two-step)

1. `img2img_describe` with `{ inputImageUrl, triggerWord, lookDescription? }` → `describeJobId`.
2. Poll describe status via `api_v1_request`: `GET /img2img/describe-status/:id` (no typed tool — synchronous in practice).
3. `img2img_generate` with `{ inputImageUrl, prompt, triggerWord, loraUrl, … }` → job id.
4. `img2img_status` until `completed`/`failed`.

#### `img2img_describe`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. At minimum `inputImageUrl` (or `inputImageBase64`) and `triggerWord`. Free — no credits. |

`POST /img2img/describe`. Returns `{ describeJobId }`. Poll with `api_v1_request` `GET /img2img/describe-status/:describeJobId`.

#### `img2img_generate`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Source image + character LoRA fields. Omit `prompt` to run describe internally. |

`POST /img2img/generate`. Returns a job id; poll `img2img_status`.

#### `img2img_status`

| Parameter | Type | Description |
|-----------|------|-------------|
| `jobId` | string | Required |

`GET /img2img/status/:jobId` — job status and `outputUrl` when complete.

---

### GPT-X (`gptx_*`)

Conversational prompt assistant — enhances natural-language requests into generation-ready prompts. **Does not generate media itself.** See **`docs/public-api/18-gptx.md`**.

#### Typical sequence

1. `gptx_send` with `{ message, conversationId?, modelId?, modelName?, isNsfw?, referenceImageUrl? }` → `enhancedPrompt`, `aspectRatio`, `conversationId`, `aiMessageId`.
2. Call a generation tool (`generate_recreate`, `mcx_generate`, `creator_studio_image`, …) with the enhanced prompt.
3. Attach the result to the conversation via `api_v1_request` `PATCH /gptx/messages/:aiMessageId` with `{ generationId }`.
4. `gptx_conversations` or `api_v1_request` `GET /gptx/conversations/:id` to render the thread.

GPT-X calls are free; credits apply only to downstream generation.

#### `gptx_conversations`

No parameters. `GET /gptx/conversations` — list saved assistant conversations.

#### `gptx_send`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. At minimum `message`. Optional: `conversationId`, `modelId`, `modelName`, `isNsfw`, `referenceImageUrl`. |

`POST /gptx/send` — synchronous. Returns enhanced prompt and conversation metadata.

---

### Utility tools

Standalone media utilities. See **`docs/public-api/19-tools.md`**.

#### `upscale_image`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. `inputImageUrl` (public HTTPS URL) or reference a prior generation via `generationId`. |

`POST /upscale` — async. Returns a `generationId`. Poll with `wait_for_generation` or `api_v1_request` `GET /upscale/status/:generationId`. Credits refunded on failure.

#### `synthid_remove`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Image URL or generation reference. |

`POST /synthid-remove` — async watermark removal. Poll with `wait_for_generation`.

#### `video_repurpose_generate`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Batch video repurposing config (source videos, output presets). |

`POST /video-repurpose/generate-with-worker` — async batch job. Returns a `jobId`.

#### `video_repurpose_job`

| Parameter | Type | Description |
|-----------|------|-------------|
| `jobId` | string | Required |

`GET /video-repurpose/jobs/:jobId` — job status and output URLs.

---

### Flow Studio (`flows_*`)

Visual pipeline builder: define a graph of typed nodes connected by edges, then execute server-side. See **`docs/public-api/20-flows.md`**.

#### Typical sequence

1. `flows_node_types` — discover node types, ports, default config, and per-node credit costs.
2. Build a `{ name, nodes[], edges[] }` document (see public-api flows doc for port/handle rules).
3. `flows_create` with `{ body: flowDocument }` → `flowId`.
4. `flows_run` with `{ flowId, body? }` → `{ runId, status: "pending" }`.
5. Poll `flows_run_status` with `{ runId }` every few seconds until `status` is `completed`, `failed`, or `cancelled`. Read outputs from `nodeResults`.

#### `flows_list`

No parameters. `GET /flows` — saved pipelines (up to 100, most recently updated).

#### `flows_node_types`

No parameters. `GET /flows/node-types` — full node type catalog with input/output ports and `defaultData`.

#### `flows_get`

| Parameter | Type | Description |
|-----------|------|-------------|
| `flowId` | string | Required |

`GET /flows/:id` — flow document (nodes + edges) plus recent run summaries.

#### `flows_create`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. `name`, `description`, `nodes`, `edges`, optional `thumbnail`, `isPublic`. |

`POST /flows` — creates a flow. Returns `{ flow: { id, … } }`.

#### `flows_run`

| Parameter | Type | Description |
|-----------|------|-------------|
| `flowId` | string | Required |
| `body` | object | Optional run overrides |

`POST /flows/:id/run` — fire-and-forget background execution. Returns `{ runId, status: "pending" }`.

#### `flows_run_status`

| Parameter | Type | Description |
|-----------|------|-------------|
| `runId` | string | Required |

`GET /flows/runs/:runId` — canonical poll target. Returns `status`, `creditsUsed`, and per-node `nodeResults`.

**Additional flow routes** (no typed tools — use `api_v1_request`): `PUT /flows/:id`, `DELETE /flows/:id`, `GET /flows/:id/runs`, `DELETE /flows/runs/:runId` (cancel), `POST /flows/estimate-credits`.

#### SSE live stream (direct HTTP only)

MCP has **no SSE tool**. For real-time flow progress outside MCP, open a direct REST connection:

```
GET /api/v1/flows/runs/:runId/stream
```

- Content-Type: `text/event-stream`
- Same API-key auth as other endpoints
- Pushes `log`, `node`, and `flow` events as JSON in SSE `data:` frames
- Heartbeat comment lines every ~20s — ignore them
- Connection lifetime is capped (~25s default); server sends a `reconnect` event and closes — **reconnect immediately** or fall back to `flows_run_status` polling
- If the run is already terminal when you connect, one final payload is sent and the stream closes

**In MCP agents:** prefer `flows_run_status` polling (every 3–5s). SSE is for custom HTTP clients and the web UI, not MCP tool calls.

---

### Community gallery (`gallery_*`)

Browse and interact with the public SFW community feed. See **`docs/public-api/21-gallery.md`**.

**Note:** You cannot post directly to the gallery. Content enters when an eligible SFW generation is submitted with `shareWithCommunity: true` on the generation request. A username (`PUT /me/username` via `api_v1_request`) is required before sharing.

#### `gallery_feed`

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int 1–100 | Optional page size |
| `cursor` | string | Optional pagination cursor |
| `sort` | string | Optional — `popular`, `recent`, `oldest` |

`GET /public-gallery` — community feed. Works without auth; authenticated calls include `likedByMe`.

#### `gallery_post`

| Parameter | Type | Description |
|-----------|------|-------------|
| `postId` | string | Required |

`GET /public-gallery/:postId` — one post with media, prompt, and engagement counts.

#### `gallery_like`

| Parameter | Type | Description |
|-----------|------|-------------|
| `postId` | string | Required |

`POST /public-gallery/:postId/like` — toggle like (idempotent toggle).

#### `gallery_comment`

| Parameter | Type | Description |
|-----------|------|-------------|
| `postId` | string | Required |
| `text` | string | Required comment body |

`POST /public-gallery/:postId/comments` — add a comment.

**Additional gallery routes** (via `api_v1_request`): `PUT /me/username`, share-as-video, list/delete own publications — see OpenAPI resource.

---

### API keys (no typed tools)

Integrator API key CRUD is on the product surface but has **no dedicated MCP tools**. Use `api_v1_request`:

| Action | Call |
|--------|------|
| List keys | `GET /user/api-keys` |
| Create key | `POST /user/api-keys` with `{ name?, scopes?, defaultWebhookUrl? }` |
| Update key | `PATCH /user/api-keys/:keyId` — rename, scopes, default webhook |
| Delete key | `DELETE /user/api-keys/:keyId` |
| Rotate secret | `POST /user/api-keys/:keyId/regenerate` |

All are synchronous. The raw `mcl_…` secret is returned only on create/regenerate — store it immediately. Body shapes: `modelclone://v1/openapi` paths under `/user/api-keys`.

**Do not delete the key currently bound to your MCP session** — subsequent tool calls will fail with `401`.

---

### When to use typed tools vs `api_v1_request`

| Situation | Prefer |
|-----------|--------|
| Common product action with a typed tool | Typed tool (`mcx_generate`, `flows_run`, …) |
| Route has no typed tool (describe-status, flow cancel, GPT-X message patch, API keys) | `api_v1_request` |
| Unsure a path exists | Read `modelclone://v1/route-catalog` first |
| Unsure of JSON body shape | Read `modelclone://v1/openapi` first |


---

## 08. Resources {#08-resources}

## MCP resources

Resources are **read-only context** the MCP client can fetch and inject into the agent's working memory. They do not execute API calls or charge credits.

| URI | MIME | Content |
|-----|------|---------|
| `modelclone://v1/route-catalog` | `application/json` | Allowed integrator routes (method + path under `/api/v1`) with rate-limit buckets |
| `modelclone://v1/openapi` | `text/yaml` | Full OpenAPI 3 contract for `/api/v1` (request/response shapes) |
| `modelclone://v1/base` | `text/plain` | REST base URL + auth header hints |
| `modelclone://v1/getting-started` | `text/markdown` | Typical workflows and tool sequencing |

### How to read resources

MCP clients expose resources through the standard MCP protocol:

1. **`resources/list`** — discover available URIs (or rely on the table above).
2. **`resources/read`** with `{ uri: "modelclone://v1/route-catalog" }` — fetch the content.

In Claude.ai / Cursor / Claude Desktop, resources are often loaded automatically when the connector initializes, or on demand when the agent references them. The health endpoint (`GET https://mcp.modelclone.app/mcp/health`) also lists resource URIs without auth.

Each resource returns a single text payload suitable for grounding — JSON, YAML, plain text, or markdown depending on the URI.

### `modelclone://v1/route-catalog`

**When to read:** before calling `api_v1_request` with an unfamiliar path, or when exploring what the integrator surface covers.

Contains:

- `integratorProductCount` — number of allowed routes
- `routes[]` — each with `method`, `path`, `auth`, `providerBucket`, `completion` (`sync` | `poll`), `apiKeyLimitPerMin`
- `sessionOnlyNamespaces` — path prefixes where API keys get `403 SESSION_ONLY_ENDPOINT` (billing, admin, referrals, …)
- `apiKeyLimitsPerMin` — per-bucket submit limits (`standard`, `advanced`, `dedicated`, `cinematic`, `repurposer`, …)
- `flowStudio` — metadata about Flow Studio route availability

Use to answer: *"Is `POST /flows/:id/run` allowed?"*, *"Does this route need polling?"*, *"Which rate-limit bucket applies?"*

**Does not contain:** request body field definitions — use the OpenAPI resource for those.

### `modelclone://v1/openapi`

**When to read:** before constructing `body` or `options` for any tool, especially `api_v1_request` and tools that accept free-form `body` (`mcx_generate`, `flows_create`, `creator_studio_image`, …).

The full generated `docs/openapi/v1.openapi.yaml` — request/response schemas, enums, required fields, and error shapes for every integrator route.

Use to answer: *"What fields does `POST /modelclone-x/generate` accept?"*, *"What does a 402 response look like?"*

**Fallback:** if the YAML is not bundled in your deployment, `GET /api/v1/openapi.yaml` over REST returns the same spec.

### `modelclone://v1/base`

Quick reference for direct HTTP outside MCP:

```
baseUrl: https://modelclone.app/api/v1
auth: X-Api-Key: mcl_… or Authorization: Bearer mcl_…
```

Use when the agent needs to construct a raw `curl`, open an SSE stream (flow runs), or upload via presigned URLs.

### `modelclone://v1/getting-started`

Curated workflow guide — recommended tool order for model creation, generation, NSFW video sessions, and Flow Studio. Good first read in a new session after `get_me`.

### When to read resources vs convenience tools

| Need | Use | Why |
|------|-----|-----|
| Confirm auth + credits | **`get_me` tool** | Live account state, not static docs |
| Live credit costs | **`get_pricing_generation` tool** | Authoritative pricing, changes without redeploy |
| Run a known product action | **Typed tool** (`mcx_generate`, `gallery_feed`, `flows_run`, …) | One call, validated params, no path guessing |
| Poll async job to completion | **`wait_for_generation`** or feature status tool | Server-side polling built in |
| Explore whether a route exists | **Resource `route-catalog`** | Complete allowlist for `api_v1_request` |
| Learn JSON body field names/types | **Resource `openapi`** | Full schema; too large to memorize |
| Route with no typed tool (API keys, flow cancel, img2img describe-status) | **`api_v1_request`** after reading catalog + openapi | Escape hatch for the long tail |
| Workflow / sequencing guidance | **Resource `getting-started`** | Narrative recipes |
| Real-time flow SSE progress | **Direct HTTP** to `/flows/runs/:runId/stream` | MCP has no streaming tool — poll `flows_run_status` instead |

#### Decision flow

```
Need to call the API?
├─ Is there a typed tool? ──yes──► use typed tool
└─ no ──► read route-catalog (path allowed?)
          └─ read openapi (body shape)
              └─ api_v1_request
```

**Anti-pattern:** reading the entire OpenAPI spec into context on every turn. Fetch it once when building an unfamiliar payload, or grep for the specific path/method you need.

**Anti-pattern:** using `api_v1_request` for actions that have typed tools (`mcx_generate`, `upscale_image`, …) — typed tools are shorter, self-documenting, and less error-prone.


---

## 09. Claude.ai connector {#09-claude-ai-connector}

### Claude.ai custom connector

#### Prerequisites

- [ ] `curl https://mcp.modelclone.app/mcp/health` returns **JSON** (not HTML)
- [ ] You have an **`mcl_`** key (Business or partner access)
- [ ] Vercel deploy includes `vercel.json` rewrite → `/api/index.js`

#### Step-by-step (copy-paste)

1. Open **[claude.ai](https://claude.ai)** → **Settings** → **Connectors** → **Add custom connector**.
2. Fill in:

| Field | Value |
|-------|--------|
| **Name** | `ModelClone` (any label) |
| **URL** | `https://mcp.modelclone.app/mcp` |
| **Transport** | Streamable HTTP (MCP) |
| **Authentication** | OAuth — click **Connect** (browser login on modelclone.app) |

3. Save and enable the connector for your chats.

Fallback: **Request headers** → `x-api-key` = your `mcl_…` if OAuth UI is unavailable.

Do **not** use `https://modelclone.app/mcp` — MCP is only on the **`mcp.`** subdomain.

#### Verify before connecting

```bash
curl -sS https://mcp.modelclone.app/mcp/health | head -c 200
## Expect JSON with logoUrl + icons (ModelClone mark for connector UI)
```

Expect JSON starting with `{"name":"modelclone-mcp"`. HTML = broken Vercel routing (§2).

#### Smoke test in Claude

Start a new chat with the connector enabled. Ask:

> Use the ModelClone MCP `get_me` tool and tell me my email and total credits.

Expected: successful tool call with your account data.

#### Second test (async generation)

> Call `enhance_prompt` with prompt "sunset rooftop portrait" and mode "casual", then show me the enhanced text.

Expected: synchronous response with `enhancedPrompt` (5 credits default).

#### Session behavior (automatic)

Claude handles MCP sessions for you:

- First tool call creates a session; **`mcp-session-id`** is managed by the client.
- **30-minute idle** → session expires; Claude reconnects on the next tool call.
- **Do not paste a second key** into the connector while a chat is active — session binding rejects key swaps (§3).

#### Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Connector won't connect | DNS / wrong URL | Use exact URL above; check health JSON |
| HTML in health | Bad Vercel rewrite | Redeploy with `/api/index.js` host rule (§2) |
| 401 invalid key | Revoked / typo | Regenerate key in **Settings → API** |
| 403 session bound to different key | Key changed mid-session | Disable/re-enable connector with one key |
| 410 `API_V1_SUNSET` | V1 API retired | Read `v2BaseUrl` in error; migrate connector |
| 403 on generate | Insufficient credits or plan | `get_me` → check `totalCredits` |
| 429 | Rate limit | Wait `Retry-After`; slow down submits |
| Session errors after idle | 30 min expiry or cold start | Retry — Claude usually re-initializes |

#### Security

- Treat the connector key like a password — never commit it or paste into public chats.
- Rotate if exposed (Settings → API → regenerate).
- Claude sends the key to `mcp.modelclone.app` only — upstream calls go to `modelclone.app/api/v1`.
- Do not delete the API key bound to your active connector without updating the connector first.


---

## 10. Cursor and Claude Desktop {#10-cursor-and-claude-desktop}

### Claude Code (recommended — remote HTTP)

From any terminal with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed:

```bash
claude mcp add --transport http modelclone https://mcp.modelclone.app/mcp \
  --header "Authorization: Bearer mcl_YOUR_KEY_HERE"
```

Replace `mcl_YOUR_KEY_HERE` with your 44-character integrator key.

Verify:

```bash
claude mcp list
```

In Claude Code, ask: *"Call ModelClone `get_me` and show my credits."*

Remove if needed:

```bash
claude mcp remove modelclone
```

#### Session notes

Claude Code manages `mcp-session-id` automatically. If you see **"Session bound to a different API key"**, remove and re-add the server with one key. Idle sessions expire after **30 minutes**.

---

### Cursor (remote HTTP — recommended)

Create or edit **`.cursor/mcp.json`** in your project root, or **`~/.cursor/mcp.json`** globally:

```json
{
  "mcpServers": {
    "modelclone": {
      "url": "https://mcp.modelclone.app/mcp",
      "headers": {
        "Authorization": "Bearer mcl_YOUR_KEY_HERE"
      }
    }
  }
}
```

Alternative header (equivalent):

```json
"headers": {
  "X-Api-Key": "mcl_YOUR_KEY_HERE"
}
```

1. Save the file.
2. **Cursor Settings → MCP** — confirm `modelclone` appears and is enabled.
3. Reload window (**Developer: Reload Window**).
4. Confirm tools: `get_me`, `api_v1_request`, `wait_for_generation`, …

---

### Cursor / Claude Desktop (local stdio)

Use when HTTP MCP is unavailable or for offline dev. See §5 for install steps.

**`.cursor/mcp.json` or Claude Desktop config:**

```json
{
  "mcpServers": {
    "modelclone": {
      "command": "node",
      "args": ["C:/Users/YOU/path/to/modelclone/integrations/mcp-modelclone/src/index.mjs"],
      "env": {
        "MODELCLONE_API_KEY": "mcl_YOUR_KEY_HERE",
        "MODELCLONE_BASE_URL": "https://modelclone.app"
      }
    }
  }
}
```

**Windows Claude Desktop path:** `%APPDATA%\Claude\claude_desktop_config.json`  
**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`

Use **absolute paths** to `index.mjs`. Restart the app after edits.

---

### Claude.ai (web)

Not an IDE — use the custom connector (§9). Same URL and bearer token as Cursor HTTP above.

---

### Verify connectivity (all clients)

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

No auth required. Expect `"version": "2.1.0"` and `"toolGroups"`.

Test REST + key separately:

```bash
curl -sS -H "X-Api-Key: mcl_YOUR_KEY_HERE" https://modelclone.app/api/v1/me
```

---

### Tips for agents

- Start every session with **`get_me`** and **`get_pricing_generation`**.
- Read **`modelclone://v1/route-catalog`** when unsure which path to call via `api_v1_request`.
- After async submits, use **`wait_for_generation`** — do not spam POST submits (rate limits).
- Poll every **5–10 s**, not sub-second.
- One in-flight generation when testing to avoid `GENERATION_QUEUE_FULL`.


---

## 11. Security and route allowlist {#11-security-and-route-allowlist}

### Route allowlist

MCP tools **cannot** call arbitrary URLs. `api_v1_request` only allows routes present in:

`docs/generated/V1_ROUTE_INVENTORY.json` where `"integratorProduct": true`.

Source of truth is regenerated by `npm run docs:registry` from Express route files (~**100 routes**).

#### How matching works

1. **Normalize path** — strip `/api/v1` prefix if present; no trailing slash; reject `..` and `//`.
2. **Exact match** — static paths like `GET /me`, `POST /generate/free`.
3. **Parameterized match** — Express `:param` segments become regex (e.g. `GET /generations/:id` matches `/generations/abc-123`).
4. **Rejection** — throws before upstream fetch:

```
Route not on integrator product surface: POST /admin/stats. Use resource modelclone://v1/route-catalog.
```

Tool result shape: `{ ok: false, error: "<message>" }` (no HTTP round-trip).

#### Discover allowed routes

| Resource | Use |
|----------|-----|
| `modelclone://v1/route-catalog` | Full JSON inventory (methods, paths, rate-limit buckets) |
| `modelclone://v1/openapi` | Request/response body schemas |
| `GET …/mcp/health` | Tool group manifest (no auth) |

### Blocked categories

| Category | Example paths | MCP |
|----------|---------------|-----|
| Admin | `/admin/stats`, `/admin/users/...` | Not in allowlist |
| Session-only | Billing, referrals, DSAR | Listed in `sessionOnlyNamespaces` — not callable with `mcl_` |
| Cron / debug | `/cron/...`, `/debug/...` | Excluded from integrator surface |
| Internal callbacks | Provider webhooks, worker progress | Excluded |

**Flow Studio** (`/flows/*`) **is allowed** — use typed `flows_*` tools or `api_v1_request`.

If you need operator actions, use the **web admin dashboard** (session JWT), not MCP.

### API key on REST (defense in depth)

Even if a path were mis-listed:

- `/admin/*` routes use `adminMiddleware` → `403` `ADMIN_SESSION_ONLY` for `mcl_` keys on REST.
- MCP allowlist prevents calling them from tools in the first place.

### Response sanitization

API-key and MCP responses pass through `api-key-response-sanitizer.middleware.js`:

- Internal engine/provider fields stripped or renamed on generation rows.
- NSFW video pipeline hides internal prompts on generation objects (session state retains user edit text).

Integrator docs and MCP resources do **not** embed internal provider names.

### Data scope

- Each key acts as **the owning user** — same rows, credits, and generations as the web app.
- No cross-tenant access.
- Revoked keys fail immediately on the next request.

### CORS and origins

Remote MCP only accepts browser origins from Claude/Anthropic (see §4). Server-to-server clients use non-browser requests (no `Origin` header) — allowed.

### V1 sunset guard

When `api_v1_sunset` feature flag is **on**:

- `/mcp` tool traffic → **410** `API_V1_SUNSET` + `v2BaseUrl`
- `/api/v1` with `mcl_` key → same 410 body
- SPA cookie auth on `/api/v1` → **unaffected**
- `/mcp/health` → **unaffected** (discovery)

### Secrets in tool args

- Do not pass plaintext `mcl_` keys in JSON `body` fields.
- Store `integratorWebhookSecret` only on your server; include in generate POST bodies when needed.

### Recommended rotation

Rotate `mcl_` keys if exposed in chat logs, commits, or connector screenshots. After rotation, update every connector (Claude.ai, Cursor, Claude Code) — old sessions bound to the revoked key will fail with **401**.


---

## 12. Troubleshooting {#12-troubleshooting}

### Health check

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

| Response | Meaning |
|----------|---------|
| JSON with `"name": "modelclone-mcp"` | OK |
| HTML (ModelClone lander) | Vercel routing broken — see §2 |
| Connection refused / NXDOMAIN | DNS not propagated |

---

### Authentication & sessions

| Symptom | HTTP | Fix |
|---------|------|-----|
| Missing API key | 401 | Add `X-Api-Key: mcl_…` or `Authorization: Bearer mcl_…` on **every** MCP HTTP request |
| Invalid or revoked key | 401 | Mint new key in **Settings → API**; update all connectors |
| Session bound to different API key | 403 | Reconnect with one key — don't swap keys mid-session |
| Session stops after ~30 min idle | (transport error) | Re-initialize — normal; DB state preserved |
| Session lost after deploy / cold start | (transport error) | Reconnect — in-memory sessions don't survive lambda cold start |

#### Session binding example

```
403 { "error": { "code": -32001, "message": "Session bound to a different API key" } }
```

Cause: `mcp-session-id` from key A reused with key B. Fix: disconnect connector, ensure one key everywhere.

---

### V1 sunset (410)

When the v1 integrator API is retired:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "The ModelClone v1 API has been retired… (v2: https://…)",
    "data": {
      "success": false,
      "code": "API_V1_SUNSET",
      "message": "The ModelClone v1 API has been retired. Please migrate to the v2 API.",
      "v2BaseUrl": "https://…"
    }
  },
  "id": null
}
```

HTTP status: **410 Gone**. Migrate MCP connector URL and REST base to **`v2BaseUrl`**. Watch `docs/API_CHANGELOG.md`.

Stdio clients still call `/api/v1` upstream — they hit the same 410 when the flag is on.

---

### Allowlist / `api_v1_request`

| Message | Fix |
|---------|-----|
| `Route not on integrator product surface: …` | Path not on product surface — read `modelclone://v1/route-catalog` |
| `Invalid path` | No `..` or `//`; use paths like `/me` not full URLs |
| Admin or billing routes | Use web app (session auth), not MCP |

---

### Tool / REST errors (inside tool result)

MCP forwards upstream JSON in `{ httpStatus, ok, method, path, body }`. Read `body.message`, `body.code`, or `body.error`.

| Code / message | Fix |
|----------------|-----|
| `API_KEY_REQUIRES_BUSINESS_PLAN` | Upgrade or request partner override |
| `httpStatus: 402/403` credits | `get_me` — buy credits or reduce job cost |
| `httpStatus: 429` | Slow down; respect per-key capacity buckets |
| `API_KEY_PROVIDER_RATE_LIMIT` | Bucket exceeded — wait and retry |
| `GENERATION_QUEUE_FULL` | Too many in-flight jobs; wait and retry |
| `MODEL_GENDER_REQUIRED` | Set gender on model or in `modelLooks` when `enhance: true` |
| `ADMIN_SESSION_ONLY` | Route requires web session, not API key |

Rate limit buckets (submits/min per key): `standard` 30, `advanced` 30, `dedicated` 60, `cinematic` 30, `repurposer` 100 — see `docs/public-api/04-rate-limits.md`.

---

### Polling

| Symptom | Fix |
|---------|-----|
| Stuck `processing` | Keep polling `wait_for_generation` or `get_generation`; video jobs can take several minutes |
| Poll returns 429 | Increase interval to 10–15 s |
| `failed` immediately | Read `errorMessage` — often validation or content policy |
| `{ timedOut: true }` from `wait_for_generation` | Job still running — call again with same `generationId` |

---

### Stdio-specific

| Symptom | Fix |
|---------|-----|
| Process exits code 1 | `MODELCLONE_API_KEY` unset or missing `mcl_` prefix |
| Module not found | Run `npm install` in `integrations/mcp-modelclone` |
| Wrong Node version | Use Node 18+ |
| Tools empty after reload | Check stderr; verify absolute path in `args` |

---

### IDE connector issues

| Client | Symptom | Fix |
|--------|---------|-----|
| Claude.ai | Connector grey / disconnected | Re-save URL + key; verify health JSON |
| Cursor | MCP server red | Check `.cursor/mcp.json` syntax; reload window |
| Claude Code | `claude mcp list` empty | Re-run `claude mcp add` with `--transport http` |
| Any HTTP client | CORS error in browser | Use server-side or Claude-hosted origin only |

---

### Diagnostic order

1. **Health:** `curl …/mcp/health` → must be JSON.
2. **REST:** `curl -H "X-Api-Key: mcl_…" https://modelclone.app/api/v1/me` → must return your profile.
3. **MCP fails, REST OK** → routing/deploy issue on `mcp.modelclone.app` (§2).
4. **Both fail** → key, plan, or account issue.
5. **410 on both** → V1 sunset active — migrate to v2.

---

### Still stuck?

Open an issue with: health curl output (redact nothing from health), HTTP status from a failed tool call, client name (Claude.ai / Cursor / Claude Code), and whether you use HTTP or stdio.


---

## 13. Recipes {#13-recipes}

### Recipe A — Verify account

**Tools:** `get_me` → `get_pricing_generation`

1. `get_me` — confirm email, `totalCredits`, `subscriptionTier`.
2. `get_pricing_generation` — cache costs for later submits.

No credits charged.

---

### Recipe B — Creator Studio image (typed tool)

**Tools:** `creator_studio_image` → `wait_for_generation`

**Submit:**

```json
{
  "prompt": "minimal product shot, white background",
  "generationModel": "nano-banana-pro",
  "aspectRatio": "1:1",
  "resolution": "2K",
  "numImages": 1,
  "enhancePrompt": true,
  "mode": "product_shot"
}
```

**Poll:** `wait_for_generation` with `body.generation.id` from the tool result (typically 30–90 s).

Credits: `creatorStudio1K2K` default **15**/image at 2K — confirm via `get_pricing_generation`.

---

### Recipe B1 — Marketplace full set

**Tools:** `creator_studio_config` → `creator_studio_marketplace` → `wait_for_generation` for every returned id

1. Quote `creatorStudioGptImage2 × 13 + enhancePromptDefault` and get approval.
2. Submit:

```json
{
  "prompt": "premium skincare serum marketplace listing",
  "scope": "full-set",
  "referencePhotos": ["https://cdn.example.com/serum.jpg"],
  "productContext": "30 ml frosted-glass dropper bottle, gold cap",
  "brandContext": "clinical white and sage green"
}
```

3. Read `generations[]`; each row has `asset`, `id`, and `aspectRatio`.
4. Call `wait_for_generation` for each id and return URLs labeled by asset.

For a safe live smoke test, use `scope: "main"` (one image). A `207` response is partial success: keep polling successful ids and report `errors[]`.

---

### Recipe B2 — Recreate a reference photo

**Tools:** `generate_recreate` → `wait_for_generation`

**Prerequisites:** model with 3 reference photos (`get_model`).

```json
{
  "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "options": {
    "sourceImageUrl": "https://cdn.example.com/inspo/rooftop-pose.jpg",
    "outfitMode": "model",
    "extraGuidance": "golden hour warmth"
  }
}
```

Poll `body.generation.id`. Credits: `recreateImage` default **10**/image (or **16** with `genModel: "nano-banana-pro"`).

---

### Recipe B3 — Free prompt with aspect ratio

**Tools:** `generate_free` → `wait_for_generation`

```json
{
  "prompt": "candid mirror selfie, soft morning light",
  "options": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "genModel": "nano-banana-pro",
    "aspectRatio": "9:16",
    "resolution": "2K",
    "enhance": true
  }
}
```

When `enhance: true`, model gender must be set or the API returns `MODEL_GENDER_REQUIRED`.

---

### Recipe B4 — Motion video from still + driving clip

**Tools:** `generate_motion_video` → `wait_for_generation`

```json
{
  "body": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "imageUrl": "https://cdn.modelclone.app/generations/still.png",
    "videoUrl": "https://cdn.example.com/uploads/dance.mp4",
    "duration": 8,
    "prompt": "natural cinematic motion"
  }
}
```

Poll `body.generationId`. Credits: `motionXPerSec` default **9.5**/s × duration (8 s → **76** credits).

---

### Recipe C — ModelClone-X txt2img

**Tools:** `mcx_generate` → `wait_for_generation`

**Submit:**

```json
{
  "body": {
    "prompt": "portrait photo, neutral background",
    "aspectRatio": "1:1",
    "qty": 1,
    "preOptimized": true,
    "useCustomPrompt": true
  }
}
```

**Poll:** `generationIds[0]` from response → `wait_for_generation`.

Typical time: 2–5 minutes.

---

### Recipe D — NSFW image (classic LoRA)

**Prerequisites:** trained LoRA (`nsfw_training_status` → `status: "ready"`, `nsfwUnlocked: true`). LoRA setup via `api_v1_request` — [NSFW Studio](../../public-api/14-nsfw.md#lora-training).

**Tools:** `nsfw_generate` → `wait_for_generation`

```json
{
  "modelId": "<uuid>",
  "prompt": "<triggerWord> scene description…",
  "options": {
    "quantity": 1,
    "resolution": "768x1344",
    "integrationCallbackUrl": "https://api.example.com/hooks/modelclone",
    "integratorWebhookSecret": "whsec_…"
  }
}
```

Poll `generation.id`. Typical time: 1–3 minutes. Default **30** credits ( **50** for `quantity: 2`).

---

### Recipe D2 — NSFW v2 preset still

**Prerequisites:** NSFW-verified model with three reference photos.

**Tools:** `api_v1_request` (`POST /nsfw-v2/presets`) → `wait_for_generation` (per id)

```json
{
  "body": {
    "modelId": "<uuid>",
    "presetId": "lt_01_black_lace_bed",
    "aspectRatio": "9:16",
    "count": 1
  }
}
```

Preset catalog: 222 ids — category prefixes in [NSFW Studio](../../public-api/14-nsfw.md#preset-stills). Default **6** credits/image.

---

### Recipe D3 — NSFW video preset session (preview → select → render)

**Tools:** `nsfw_video_presets` → `nsfw_video_create_session` → `nsfw_video_get_session` (poll) → `nsfw_video_session_action`

**1. List presets** — call `nsfw_video_presets`; save a preset **`id`** (not `key`) and `durationSeconds`.

**2. Create session**

```json
{
  "body": {
    "modelId": "<uuid>",
    "mode": "preset",
    "presetId": "cpre_01hxyz"
  }
}
```

**3. Poll previews** — `nsfw_video_get_session` every 3–5 s until `previewImageUrls.length === 3`.

**4. Select preview**

```json
{
  "sessionId": "cnvs_01hxyz",
  "action": "select-preview",
  "body": { "previewUrl": "https://cdn.modelclone.app/…/preview-1.png" }
}
```

**5. (Optional) Edit frame** — 10 credits; poll session until `currentFrameUrl` updates:

```json
{
  "sessionId": "cnvs_01hxyz",
  "action": "edit-frame",
  "body": { "prompt": "remove the necklace, keep everything else identical" }
}
```

**6. Approve → submit**

```json
{ "sessionId": "cnvs_01hxyz", "action": "approve", "body": {} }
```

```json
{ "sessionId": "cnvs_01hxyz", "action": "submit", "body": {} }
```

**7. Final poll** — `nsfw_video_get_session` every 5–10 s until `status === "completed"` → `wait_for_generation(finalGenerationId)`.

Submit cost: `ceil(sourceVideoDurationSeconds × nsfwVideoPerSec)` credits (default 78.75/s). First preview batch is **free**; regenerate previews **20** credits.

**Sanitization:** pipeline generation rows return `prompt: null` over API key — rely on session state and `outputUrl`.

Full REST detail: [NSFW Video Sessions](../../public-api/15-nsfw-video.md).

---

### Recipe E — Enhance prompt then free generate

**Tools:** `enhance_prompt` (sync) → `generate_free` → `wait_for_generation`

**Step 1 — enhance (5 credits default):**

```json
{
  "prompt": "sunset rooftop portrait",
  "options": {
    "mode": "casual",
    "genModel": "nano-banana-pro",
    "modelLooks": { "gender": "female" }
  }
}
```

**Step 2 — generate with enhanced text (`enhance: false` to avoid double-charging):**

```json
{
  "prompt": "<body.enhancedPrompt from step 1>",
  "options": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "genModel": "nano-banana-pro",
    "aspectRatio": "3:4",
    "enhance": false
  }
}
```


### Recipe F — Create a model end-to-end (wizard niche path)

**Tools:** `get_me` → `list_models` → `wizard_look_variants` → `wizard_preview_images` → `wizard_finalize_poses` → `models_status` → `get_model`

**Prerequisites:** `canCreateMore: true` from `list_models`. Wizard path is **free** (no credits on look variants, previews, or finalize).

#### 1. Confirm account

```json
{}
```

Call `get_me` — note `totalCredits` (only needed if you later use the classic paid path).

#### 2. Check model slot

Call `list_models` with no parameters. Abort if `canCreateMore` is false.

#### 3. Generate look variants

```json
{
  "body": {
    "gender": "female",
    "age": 24,
    "nicheName": "Fitness",
    "ethnicity": "Latina"
  }
}
```

Tool: `wizard_look_variants`. Pick one entry from `variants[]` (e.g. label `"Soft"`).

#### 4. Render preview images

```json
{
  "body": {
    "gender": "female",
    "age": 24,
    "nicheName": "Fitness",
    "variants": [
      { "label": "Soft", "looks": { "gender": "female", "ethnicity": "Latina", "hairColor": "Dark Brown", "bodyType": "Athletic" } }
    ]
  }
}
```

Tool: `wizard_preview_images`. Save `previews[0].referenceUrl` and `previews[0].looks`.

#### 5. Finalize poses (async)

```json
{
  "body": {
    "name": "FitCreator24",
    "referenceUrl": "https://cdn.modelclone.app/references/prev-1.jpg",
    "gender": "female",
    "age": 24,
    "ethnicity": "Latina",
    "hairColor": "Dark Brown",
    "bodyType": "Athletic"
  }
}
```

Tool: `wizard_finalize_poses`. Read `model.id` from the 202 response.

#### 6. Poll until ready

```json
{ "id": "550e8400-e29b-41d4-a716-446655440000" }
```

Tool: `models_status` every **3–5s** until `status === "ready"` (or `"failed"`).

Typical time: 1–3 minutes.

#### 7. Verify

```json
{ "modelId": "550e8400-e29b-41d4-a716-446655440000" }
```

Tool: `get_model` — confirm all three `photo*Url` fields and `status: "ready"`. Use `model.id` in `generate_recreate` and other generation tools.

---

#### Alternate paths (same poll target)

| Path | Submit sequence | Credits (defaults) |
|------|-----------------|-------------------|
| **Custom description** | `wizard_custom_reference` → `wizard_finalize_poses` | 0 (+ 10 per `regenerate: true` on custom reference) |
| **Upload photos** | Upload 3 URLs → `wizard_upload_save` | 0 |
| **Classic two-step** | `models_generate_reference` (150) → `models_generate_poses` (750) | 900 total |

All async paths poll `models_status` with the returned `model.id`.

---

### Pacing (avoid 429)

- Wait **6+ seconds** between POST submits on the same account.
- Poll every **10–15s**, not sub-second.
- One async job at a time when testing.


---

## 14. Limitations and REST parity {#14-limitations-and-rest-parity}

### MCP vs direct REST

| Capability | MCP | REST |
|------------|-----|------|
| Integrator routes (~100) | Yes (`api_v1_request` + typed tools) | Yes |
| Auth | `mcl_` per MCP request | `mcl_` header |
| Credits / billing | Same | Same |
| Rate limits | Same per-key buckets | Same |
| Async poll | `wait_for_generation` | `GET /generations/:id` |
| Integrator webhooks | Yes (in POST `body`) | Yes |
| OpenAPI download | Via resource `modelclone://v1/openapi` | Direct |
| Flow Studio | Typed `flows_*` tools | `/api/v1/flows/*` |

### Limitations

#### Multipart uploads

MCP tools only send **JSON**. For file uploads:

1. `api_v1_request` → `POST /upload/presign` (or integrator upload flow).
2. Upload bytes to returned URL with HTTP PUT from your environment.
3. Pass resulting HTTPS URL in generate `body` (`inputImageUrl`, `referencePhotos`, etc.).

Mask multipart for Creator Studio: upload mask to your CDN or use `POST /upload` first, then pass URL.

#### HTTP session transport (remote only)

| Limitation | Detail |
|------------|--------|
| In-memory sessions | Stored on lambda instance — not shared across Vercel instances |
| 30-minute idle TTL | Session deleted; client must re-`initialize` |
| Cold starts | Session lost; reconnect automatically |
| Key binding | One session per API key — cannot hot-swap keys |

Stdio has no HTTP session — key is fixed in `MODELCLONE_API_KEY` for the process lifetime.

#### V1 sunset

When `api_v1_sunset` flag is active, **all** MCP and `/api/v1` API-key traffic returns **410** `API_V1_SUNSET`. Plan migration before cutover; health endpoint remains for discovery.

#### SSE / streaming

- Flow run SSE (`GET /flows/runs/:runId/stream`) — direct HTTP only (~25 s connection cap). MCP clients should poll `flows_run_status`.
- MCP transport itself uses Streamable HTTP — handled by the client library.

#### Browser-only routes

Some `/auth/*` and OAuth callback routes are web-only — not useful via MCP.

#### Admin / operator

No admin surface — intentional. Admin paths are excluded from the allowlist and return `ADMIN_SESSION_ONLY` on REST even if mis-listed.

#### Typed tool coverage

**226 tools** cover all major integrator-product flows, including live Creator Studio engine discovery, enhancer preview, and marketplace sets. Remaining routes (JWT auth signup/login, Fanvue OAuth browser start, web push token CRUD, API key self-service, onboarding trial, viral-reels stream tokens) use **`api_v1_request`** with `modelclone://v1/route-catalog` — documented, not a functional gap for API-key backends.

### REST parity statement

**Any integrator workflow achievable with `curl` + `mcl_` is achievable with MCP** via typed tools and/or `api_v1_request`, subject to:

- JSON-only tool payloads (upload URLs instead of raw files).
- Client poll discipline (`wait_for_generation`, status tools).
- Account eligibility (Business / partner access).
- V1 sunset migration when v2 launches.

### Roadmap (not shipped)

- Redis-backed MCP sessions for multi-instance Vercel.
- MCP-native upload helper tool (presign + PUT guidance in one call).


---

## 15. Appendix {#15-appendix}

### Health endpoint

`GET https://mcp.modelclone.app/mcp/health` — **no authentication**.

Example response (truncated):

```json
{
  "name": "modelclone-mcp",
  "version": "2.1.0",
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
    "generate": ["generate_recreate", "generate_free", "enhance_prompt", "generate_motion_video", "creator_studio_config", "creator_studio_image", "creator_studio_enhance", "creator_studio_marketplace", "creator_studio_video"],
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

### Environment variables

#### MCP HTTP server (Vercel / Node)

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | Prisma — API key validation |
| `JWT_SECRET` | Yes | App auth (shared server) |
| `MODELCLONE_BASE_URL` | No | Upstream API origin (default `https://modelclone.app`) |
| `API_V2_BASE_URL` | No | Default `v2BaseUrl` in sunset 410 responses |
| `NODE_ENV` | No | `production` enables strict CORS |

#### Stdio client (local)

| Variable | Required | Description |
|----------|----------|-------------|
| `MODELCLONE_API_KEY` | Yes | Your `mcl_…` key |
| `MODELCLONE_BASE_URL` | No | Same as above |

### Source file map

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

### npm scripts

```bash
node scripts/merge-mcp-doc.mjs    # Build MODELCLONE_MCP.md from sections
cd integrations/mcp-modelclone && npm start   # Stdio server
```

### Related documents

| Doc | Contents |
|-----|----------|
| `docs/mcp/MODELCLONE_MCP.md` | This handbook (merged) |
| `docs/MCP.md` | Product-facing MCP quickstart + tool reference |
| `docs/public-api/30-mcp.md` | Public API doc MCP chapter |
| `docs/public-api/README.md` | Full REST schemas |
| `integrations/mcp-modelclone/README.md` | Short connector README |
| `docs/public-api/04-rate-limits.md` | Rate limit buckets |
| `docs/public-api/05-webhooks.md` | Webhook HMAC |

### Version

MCP server version: **2.1.0** (`createModelcloneMcpServer` / health). Update this appendix when bumping the version in `mcp.routes.js`.

### Quick setup reference

| Client | Config |
|--------|--------|
| Claude.ai | Settings → Connectors → URL `https://mcp.modelclone.app/mcp`, bearer `mcl_…` |
| Claude Code | `claude mcp add --transport http modelclone https://mcp.modelclone.app/mcp --header "Authorization: Bearer mcl_…"` |
| Cursor | `.cursor/mcp.json` → `"url": "https://mcp.modelclone.app/mcp"`, `"Authorization": "Bearer mcl_…"` |
| Claude Desktop | stdio `node …/integrations/mcp-modelclone/src/index.mjs` or HTTP as above |


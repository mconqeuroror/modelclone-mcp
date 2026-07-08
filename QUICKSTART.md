# ModelClone MCP — connect your account to any AI

The ModelClone MCP server lets any MCP-capable AI (Claude.ai, Claude Code, Claude Desktop, Cursor, Windsurf, custom agents) use your ModelClone account with **1:1 product parity**: create AI models, generate SFW/NSFW images and videos, run Flow Studio pipelines, browse the gallery, and poll jobs — everything the app can do, minus billing.

- **Endpoint:** `https://mcp.modelclone.app/mcp` (Streamable HTTP; also mounted at `/mcp` on the main origin)
- **Auth:** your integrator API key on every request — `X-Api-Key: mcl_…` or `Authorization: Bearer mcl_…`
- **Get a key:** app → **Settings → API** (Business plan, active or trialing; or admin-granted). Manage via `GET/POST/PATCH/DELETE /api/v1/user/api-keys`.
- **Deep-dive doc:** [MODELCLONE_MCP.md](./MODELCLONE_MCP.md) (architecture, transport, full tool reference, troubleshooting)
- **Integrator tool tables:** [TOOLS-REFERENCE.md](./TOOLS-REFERENCE.md)

> **V1 sunset:** when the v2 API launches, MCP and `/api/v1` API-key traffic return **`410 Gone`** with code **`API_V1_SUNSET`** and a **`v2BaseUrl`** pointer. Watch `docs/API_CHANGELOG.md`.

## Setup

Replace `mcl_YOUR_KEY_HERE` with your 44-character integrator key from **Settings → API**.

### Claude.ai (web connector)

**Settings → Connectors → Add custom connector:**

| Field | Value |
|-------|--------|
| URL | `https://mcp.modelclone.app/mcp` |
| Auth | Bearer token — paste full `mcl_…` key |

Do **not** use `https://modelclone.app/mcp` — only the **`mcp.`** subdomain.

### Claude Code

```bash
claude mcp add --transport http modelclone https://mcp.modelclone.app/mcp \
  --header "Authorization: Bearer mcl_YOUR_KEY_HERE"
```

Verify: `claude mcp list` · smoke test: *"Call ModelClone get_me and show my credits."*

### Cursor (HTTP — recommended)

`.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global):

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

Reload window after saving. Alternative header: `"X-Api-Key": "mcl_YOUR_KEY_HERE"`.

### Cursor / Claude Desktop (stdio — optional)

For local dev or clients without HTTP MCP:

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

Claude Desktop config: `%APPDATA%\Claude\claude_desktop_config.json` (Windows) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS). Use absolute paths.

### Verify connectivity

Health (no auth):

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

REST + key:

```bash
curl -sS -H "X-Api-Key: mcl_YOUR_KEY_HERE" https://modelclone.app/api/v1/me
```

Expect JSON from both. HTML from health = broken Vercel routing ([sections/02-deployment.md](./sections/02-deployment.md)).

## Session behavior (remote HTTP)

- First request creates an **`mcp-session-id`**; clients send it on follow-ups (handled automatically by Claude.ai, Claude Code, Cursor).
- **Sessions are bound to the API key** that initialized them. Reusing a session id with a different key → **403** `"Session bound to a different API key"`.
- **Idle sessions expire after 30 minutes.** Reconnect — generations, models, and credits persist in your account.
- **Vercel cold starts** drop in-memory sessions; clients should re-`initialize` on transport errors.

Stdio transport has no HTTP session — key is fixed in `MODELCLONE_API_KEY`.

## Route allowlist

`api_v1_request` only calls routes where `integratorProduct: true` in `modelclone://v1/route-catalog` (~100 routes). Blocked paths return `{ ok: false, error: "Route not on integrator product surface…" }` before any upstream fetch. Admin, billing, and session-only routes are excluded.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| 401 Missing API key | Add `Authorization: Bearer mcl_…` or `X-Api-Key` on every MCP request |
| 401 Invalid or revoked key | Regenerate in Settings → API; update all connectors |
| 403 Session bound to different key | Reconnect with one key — don't rotate mid-session |
| 410 `API_V1_SUNSET` | Migrate to `v2BaseUrl` from error payload |
| Route not on integrator product surface | Use paths from `modelclone://v1/route-catalog` only |
| Session stops after ~30 min | Normal idle expiry — reconnect |
| Health returns HTML | Fix `vercel.json` host rewrite → `/api/index.js` |
| MCP fails, REST `/me` OK | `mcp.modelclone.app` routing issue |
| Both fail | Key, plan, or account issue |

Full detail: [sections/12-troubleshooting.md](./sections/12-troubleshooting.md).

## Documentation map

| Doc | Audience | Contents |
|-----|----------|----------|
| **[QUICKSTART.md](./QUICKSTART.md)** (this file) | Anyone connecting an MCP client | Setup, session rules, troubleshooting, tool inventory |
| **[TOOLS-REFERENCE.md](./TOOLS-REFERENCE.md)** | Integrators / API consumers | Full per-tool parameter tables for all **110** tools |
| **[MODELCLONE_MCP.md](./MODELCLONE_MCP.md)** | Operators & agent authors | Merged deep reference — transport, security, recipes, every section |
| **[sections/07-tools-convenience.md](./sections/07-tools-convenience.md)** | Agent implementers | Canonical typed-tool reference (schemas, examples, poll flows) |
| **[sections/06-tool-api-v1-request.md](./sections/06-tool-api-v1-request.md)** | Escape-hatch users | `api_v1_request` allowlist, errors, webhook examples |
| **[sections/13-recipes.md](./sections/13-recipes.md)** | Workflow builders | End-to-end JSON walkthroughs |

MCP resources (read via `resources/read`): `modelclone://v1/route-catalog`, `modelclone://v1/openapi`, `modelclone://v1/getting-started`.

## Tool inventory

**110 tools** — 108 REST proxy tools plus `wait_for_generation` (server-side poll) and `api_v1_request` (escape hatch). Every async submit returns a generation/job/session id; finish with `wait_for_generation` or the feature status tool. (`enhance_prompt`, `describe_target`, `extract_frames` are synchronous.)

| Group | Tools |
|-------|-------|
| Meta & account | `get_me`, `get_pricing_generation`, `confirm_adult`, `get_notifications`, `notifications_mark_read`, `notifications_mark_all_read`, `get_notification_preferences`, `set_notification_preferences`, `get_my_flags`, `get_plans`, `api_v1_request` |
| Generations | `list_generations`, `get_generation`, `wait_for_generation`, `generations_batch_delete`, `generations_monthly_stats` |
| Models & wizard | `list_models`, `get_model`, `create_model`, `delete_model`, `models_generate_reference`, `models_generate_poses`, `models_status`, `wizard_*` (5) |
| SFW generate | `generate_image_identity`, `generate_recreate`, `generate_free`, `generate_preset_recreate`, `enhance_prompt`, `generate_motion_video`, `generate_video_motion`, `generate_video_directly`, `generate_face_swap_video`, `generate_image_faceswap`, `generate_complete_recreation`, `describe_target`, `extract_frames`, `generate_advanced`, `creator_studio_*` (5) |
| NSFW | `nsfw_*` (18) + `sexting_*` (6) + `nsfw_video_*` (4) |
| ModelClone-X | `mcx_config`, `mcx_generate`, `mcx_status` |
| img2img | `img2img_describe`, `img2img_describe_status`, `img2img_generate`, `img2img_status` |
| GPT-X | `gptx_conversations`, `gptx_send` |
| Utilities | `upscale_image`, `synthid_remove`, `video_repurpose_*`, `reformatter_convert`, `reformatter_status` |
| Flow Studio | `flows_*` (10) |
| Gallery & profile | `gallery_*` (7), `set_username` |
| Avatars | `avatars_*` (5) |

**Per-tool schemas, REST mappings, and JSON examples:** [TOOLS-REFERENCE.md](./TOOLS-REFERENCE.md) (integrator overview) and [sections/07-tools-convenience.md](./sections/07-tools-convenience.md) + [sections/06-tool-api-v1-request.md](./sections/06-tool-api-v1-request.md) (canonical MCP reference).

**Routes still on `api_v1_request` only:** JWT signup/login/2FA, Fanvue OAuth browser start, web push token registration, API key CRUD, viral-reels media tokens, onboarding trial routes. Read `modelclone://v1/route-catalog` before calling.

## Example workflows

| Goal | Tool chain |
|------|------------|
| Model + SFW image | `get_me` → wizard or classic model tools → `models_status` → `generate_recreate` → `wait_for_generation` |
| NSFW video preset | `nsfw_video_presets` → `nsfw_video_create_session` → poll session → `nsfw_video_session_action` chain → `wait_for_generation(finalGenerationId)` |
| Flow Studio | `flows_node_types` → `flows_create` → `flows_run` → `flows_run_status` |

Full JSON walkthroughs: [sections/13-recipes.md](./sections/13-recipes.md). Sequencing guide: resource `modelclone://v1/getting-started`.

## Limits and behavior

- Same credits and rate limits as REST — see [modelclone-api-V1](https://github.com/mconqeuroror/modelclone-api-V1) rate limits guide.
- **Session-only** (not via API key/MCP): Stripe/crypto billing, referrals, DSAR, admin.
- One MCP tool call = one REST call; nothing cached server-side.
- JSON-only payloads — upload files via presign/`POST /upload`, then pass HTTPS URLs in tool bodies.

Full limitations: [sections/14-limitations.md](./sections/14-limitations.md) · security: [sections/11-security-allowlist.md](./sections/11-security-allowlist.md).

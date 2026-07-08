# MCP resources

Resources are **read-only context** the MCP client can fetch and inject into the agent's working memory. They do not execute API calls or charge credits.

| URI | MIME | Content |
|-----|------|---------|
| `modelclone://v1/route-catalog` | `application/json` | Allowed integrator routes (method + path under `/api/v1`) with rate-limit buckets |
| `modelclone://v1/openapi` | `text/yaml` | Full OpenAPI 3 contract for `/api/v1` (request/response shapes) |
| `modelclone://v1/base` | `text/plain` | REST base URL + auth header hints |
| `modelclone://v1/getting-started` | `text/markdown` | Typical workflows and tool sequencing |

## How to read resources

MCP clients expose resources through the standard MCP protocol:

1. **`resources/list`** — discover available URIs (or rely on the table above).
2. **`resources/read`** with `{ uri: "modelclone://v1/route-catalog" }` — fetch the content.

In Claude.ai / Cursor / Claude Desktop, resources are often loaded automatically when the connector initializes, or on demand when the agent references them. The health endpoint (`GET https://mcp.modelclone.app/mcp/health`) also lists resource URIs without auth.

Each resource returns a single text payload suitable for grounding — JSON, YAML, plain text, or markdown depending on the URI.

## `modelclone://v1/route-catalog`

**When to read:** before calling `api_v1_request` with an unfamiliar path, or when exploring what the integrator surface covers.

Contains:

- `integratorProductCount` — number of allowed routes
- `routes[]` — each with `method`, `path`, `auth`, `providerBucket`, `completion` (`sync` | `poll`), `apiKeyLimitPerMin`
- `sessionOnlyNamespaces` — path prefixes where API keys get `403 SESSION_ONLY_ENDPOINT` (billing, admin, referrals, …)
- `apiKeyLimitsPerMin` — per-bucket submit limits (`standard`, `advanced`, `dedicated`, `cinematic`, `repurposer`, …)
- `flowStudio` — metadata about Flow Studio route availability

Use to answer: *"Is `POST /flows/:id/run` allowed?"*, *"Does this route need polling?"*, *"Which rate-limit bucket applies?"*

**Does not contain:** request body field definitions — use the OpenAPI resource for those.

## `modelclone://v1/openapi`

**When to read:** before constructing `body` or `options` for any tool, especially `api_v1_request` and tools that accept free-form `body` (`mcx_generate`, `flows_create`, `creator_studio_image`, …).

The full generated `docs/openapi/v1.openapi.yaml` — request/response schemas, enums, required fields, and error shapes for every integrator route.

Use to answer: *"What fields does `POST /modelclone-x/generate` accept?"*, *"What does a 402 response look like?"*

**Fallback:** if the YAML is not bundled in your deployment, `GET /api/v1/openapi.yaml` over REST returns the same spec.

## `modelclone://v1/base`

Quick reference for direct HTTP outside MCP:

```
baseUrl: https://modelclone.app/api/v1
auth: X-Api-Key: mcl_… or Authorization: Bearer mcl_…
```

Use when the agent needs to construct a raw `curl`, open an SSE stream (flow runs), or upload via presigned URLs.

## `modelclone://v1/getting-started`

Curated workflow guide — recommended tool order for model creation, generation, NSFW video sessions, and Flow Studio. Good first read in a new session after `get_me`.

## When to read resources vs convenience tools

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

### Decision flow

```
Need to call the API?
├─ Is there a typed tool? ──yes──► use typed tool
└─ no ──► read route-catalog (path allowed?)
          └─ read openapi (body shape)
              └─ api_v1_request
```

**Anti-pattern:** reading the entire OpenAPI spec into context on every turn. Fetch it once when building an unfamiliar payload, or grep for the specific path/method you need.

**Anti-pattern:** using `api_v1_request` for actions that have typed tools (`mcx_generate`, `upscale_image`, …) — typed tools are shorter, self-documenting, and less error-prone.

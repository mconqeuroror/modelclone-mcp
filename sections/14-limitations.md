## MCP vs direct REST

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

## Limitations

### Multipart uploads

MCP tools only send **JSON**. For file uploads:

1. `api_v1_request` → `POST /upload/presign` (or integrator upload flow).
2. Upload bytes to returned URL with HTTP PUT from your environment.
3. Pass resulting HTTPS URL in generate `body` (`inputImageUrl`, `referencePhotos`, etc.).

Mask multipart for Creator Studio: upload mask to your CDN or use `POST /upload` first, then pass URL.

### HTTP session transport (remote only)

| Limitation | Detail |
|------------|--------|
| In-memory sessions | Stored on lambda instance — not shared across Vercel instances |
| 30-minute idle TTL | Session deleted; client must re-`initialize` |
| Cold starts | Session lost; reconnect automatically |
| Key binding | One session per API key — cannot hot-swap keys |

Stdio has no HTTP session — key is fixed in `MODELCLONE_API_KEY` for the process lifetime.

### V1 sunset

When `api_v1_sunset` flag is active, **all** MCP and `/api/v1` API-key traffic returns **410** `API_V1_SUNSET`. Plan migration before cutover; health endpoint remains for discovery.

### SSE / streaming

- Flow run SSE (`GET /flows/runs/:runId/stream`) — direct HTTP only (~25 s connection cap). MCP clients should poll `flows_run_status`.
- MCP transport itself uses Streamable HTTP — handled by the client library.

### Browser-only routes

Some `/auth/*` and OAuth callback routes are web-only — not useful via MCP.

### Admin / operator

No admin surface — intentional. Admin paths are excluded from the allowlist and return `ADMIN_SESSION_ONLY` on REST even if mis-listed.

### Typed tool coverage

**110 typed tools** cover all major integrator-product flows. Remaining routes (JWT auth signup/login, Fanvue OAuth browser start, web push token CRUD, API key self-service, onboarding trial, viral-reels stream tokens) use **`api_v1_request`** with `modelclone://v1/route-catalog` — documented, not a functional gap for API-key backends.

## REST parity statement

**Any integrator workflow achievable with `curl` + `mcl_` is achievable with MCP** via typed tools and/or `api_v1_request`, subject to:

- JSON-only tool payloads (upload URLs instead of raw files).
- Client poll discipline (`wait_for_generation`, status tools).
- Account eligibility (Business / partner access).
- V1 sunset migration when v2 launches.

## Roadmap (not shipped)

- Redis-backed MCP sessions for multi-instance Vercel.
- MCP-native upload helper tool (presign + PUT guidance in one call).

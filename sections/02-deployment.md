## Production hosting

| Item | Value |
|------|--------|
| **Hostname** | `mcp.modelclone.app` |
| **Connector endpoint** | `https://mcp.modelclone.app/mcp` |
| **Health** | `GET https://mcp.modelclone.app/mcp/health` (no auth) |
| **App mount** | Express `app.use('/mcp', mcpRoutes)` in `src/server.js` |

## DNS

1. In your DNS provider, add **`mcp.modelclone.app`** as a **CNAME** to your Vercel project (same target as `modelclone.app`).
2. In **Vercel → Project → Domains**, add `mcp.modelclone.app` and wait for **Valid**.

## Vercel routing (critical)

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

## Verify after deploy

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

**Success** — JSON with `"name": "modelclone-mcp"`, `"version": "2.1.0"`, `"toolGroups": { … }`, `"restBase": "https://modelclone.app/api/v1"`.

**Failure** — HTML page titled "ModelClone — Cinematic AI Video…" → fix `vercel.json` and redeploy.

## Environment variables (server)

| Variable | Default | Role |
|----------|---------|------|
| `MODELCLONE_BASE_URL` | `https://modelclone.app` | Target for `api_v1_request` upstream |
| `JWT_SECRET` / DB | (required) | API key validation uses Prisma `ApiKey` table |
| `API_V2_BASE_URL` | (optional) | Default `v2BaseUrl` in V1 sunset 410 responses |

MCP does **not** use a server-side `MODELCLONE_API_KEY` — each client supplies their own `mcl_` key per request.

## V1 sunset kill-switch

Controlled by feature flag `api_v1_sunset` (admin). When active, `mcpSunsetGuard` on `POST /mcp` returns **410** before session creation. Health endpoint stays up for discovery.

## Redeploy checklist

- [ ] `vercel.json` host → `/api/index.js`
- [ ] Domain green in Vercel
- [ ] `curl …/mcp/health` returns JSON (not HTML)
- [ ] Claude connector URL = `https://mcp.modelclone.app/mcp` (not `modelclone.app/mcp`)

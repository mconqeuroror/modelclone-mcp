## Health check

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

| Response | Meaning |
|----------|---------|
| JSON with `"name": "modelclone-mcp"` | OK |
| HTML (ModelClone lander) | Vercel routing broken — see §2 |
| Connection refused / NXDOMAIN | DNS not propagated |

---

## Authentication & sessions

| Symptom | HTTP | Fix |
|---------|------|-----|
| Missing API key | 401 | Add `X-Api-Key: mcl_…` or `Authorization: Bearer mcl_…` on **every** MCP HTTP request |
| Invalid or revoked key | 401 | Mint new key in **Settings → API**; update all connectors |
| Session bound to different API key | 403 | Reconnect with one key — don't swap keys mid-session |
| Session stops after ~30 min idle | (transport error) | Re-initialize — normal; DB state preserved |
| Session lost after deploy / cold start | (transport error) | Reconnect — in-memory sessions don't survive lambda cold start |

### Session binding example

```
403 { "error": { "code": -32001, "message": "Session bound to a different API key" } }
```

Cause: `mcp-session-id` from key A reused with key B. Fix: disconnect connector, ensure one key everywhere.

---

## V1 sunset (410)

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

## Allowlist / `api_v1_request`

| Message | Fix |
|---------|-----|
| `Route not on integrator product surface: …` | Path not on product surface — read `modelclone://v1/route-catalog` |
| `Invalid path` | No `..` or `//`; use paths like `/me` not full URLs |
| Admin or billing routes | Use web app (session auth), not MCP |

---

## Tool / REST errors (inside tool result)

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

## Polling

| Symptom | Fix |
|---------|-----|
| Stuck `processing` | Keep polling `wait_for_generation` or `get_generation`; video jobs can take several minutes |
| Poll returns 429 | Increase interval to 10–15 s |
| `failed` immediately | Read `errorMessage` — often validation or content policy |
| `{ timedOut: true }` from `wait_for_generation` | Job still running — call again with same `generationId` |

---

## Stdio-specific

| Symptom | Fix |
|---------|-----|
| Process exits code 1 | `MODELCLONE_API_KEY` unset or missing `mcl_` prefix |
| Module not found | Run `npm install` in `integrations/mcp-modelclone` |
| Wrong Node version | Use Node 18+ |
| Tools empty after reload | Check stderr; verify absolute path in `args` |

---

## IDE connector issues

| Client | Symptom | Fix |
|--------|---------|-----|
| Claude.ai | Connector grey / disconnected | Re-save URL + key; verify health JSON |
| Cursor | MCP server red | Check `.cursor/mcp.json` syntax; reload window |
| Claude Code | `claude mcp list` empty | Re-run `claude mcp add` with `--transport http` |
| Any HTTP client | CORS error in browser | Use server-side or Claude-hosted origin only |

---

## Diagnostic order

1. **Health:** `curl …/mcp/health` → must be JSON.
2. **REST:** `curl -H "X-Api-Key: mcl_…" https://modelclone.app/api/v1/me` → must return your profile.
3. **MCP fails, REST OK** → routing/deploy issue on `mcp.modelclone.app` (§2).
4. **Both fail** → key, plan, or account issue.
5. **410 on both** → V1 sunset active — migrate to v2.

---

## Still stuck?

Open an issue with: health curl output (redact nothing from health), HTTP status from a failed tool call, client name (Claude.ai / Cursor / Claude Code), and whether you use HTTP or stdio.

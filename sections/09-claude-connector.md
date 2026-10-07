## Claude.ai custom connector

### Prerequisites

- [ ] `curl https://mcp.modelclone.app/mcp/health` returns **JSON** (not HTML)
- [ ] You have an **`mcl_`** key (Business or partner access)
- [ ] Vercel deploy includes `vercel.json` rewrite → `/api/index.js`

### Step-by-step (copy-paste)

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

### Verify before connecting

```bash
curl -sS https://mcp.modelclone.app/mcp/health | head -c 200
# Expect JSON with logoUrl + icons (ModelClone mark for connector UI)
```

Expect JSON starting with `{"name":"modelclone-mcp"`. HTML = broken Vercel routing (§2).

### Smoke test in Claude

Start a new chat with the connector enabled. Ask:

> Use the ModelClone MCP `get_me` tool and tell me my email and total credits.

Expected: successful tool call with your account data.

### Second test (async generation)

> Call `enhance_prompt` with prompt "sunset rooftop portrait" and mode "casual", then show me the enhanced text.

Expected: synchronous response with `enhancedPrompt` (5 credits default).

### Session behavior (automatic)

Claude handles MCP sessions for you:

- First tool call creates a session; **`mcp-session-id`** is managed by the client.
- **30-minute idle** → session expires; Claude reconnects on the next tool call.
- **Do not paste a second key** into the connector while a chat is active — session binding rejects key swaps (§3).

### Troubleshooting

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

### Security

- Treat the connector key like a password — never commit it or paste into public chats.
- Rotate if exposed (Settings → API → regenerate).
- Claude sends the key to `mcp.modelclone.app` only — upstream calls go to `modelclone.app/api/v1`.
- Do not delete the API key bound to your active connector without updating the connector first.

## API key format

- Prefix: **`mcl_`**
- Length: **44 characters** total (`mcl_` + 40 random)
- Issued in **Settings → API** (any account) — see integrator auth doc.

## Headers (every MCP HTTP request)

Send **one** of:

| Header | Example |
|--------|---------|
| `X-Api-Key` | `X-Api-Key: mcl_AbCdEf…` |
| `Authorization` | `Authorization: Bearer mcl_AbCdEf…` |
| `Authorization` | `Authorization: ApiKey mcl_AbCdEf…` |

Claude.ai connector: paste the full key in the connector **Bearer / API key** field (maps to the above).

**Do not** send a JWT `Bearer eyJ…` unless you intend session auth on non-MCP routes — integrators should use **`mcl_` only**.

## Validation flow

1. `extractApiKeyFromRequest` reads headers (`src/lib/mcp/extractApiKey.js`).
2. `validateApiKey` bcrypt-compares against `ApiKey` rows (`src/lib/mcp/validateApiKey.js`).
3. Revoked keys and `banLocked` users are rejected.
4. `lastUsedAt` is updated on success (async).

## Error responses (HTTP layer)

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

## Eligibility (same as REST)

- Mint keys in Settings — available to every account (no tier gate).
- MCP does **not** bypass plan checks — it proxies your account's real credits and limits.

## Session binding

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

## Idle session expiry

- **`SESSION_TTL_MS` = 30 minutes** — last activity timestamp refreshed on each request.
- Prune job runs every 5 minutes; expired sessions are deleted from memory.
- After expiry, send a fresh `initialize` (clients usually do this automatically).
- **No data loss** — generations, models, and credits live in the database under your account.

## V1 sunset (410)

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

## Protocol

Production MCP uses **Streamable HTTP** (`@modelcontextprotocol/sdk/server/streamableHttp.js`).

| | |
|---|---|
| **Endpoint** | `POST https://mcp.modelclone.app/mcp` (also handles session continuation) |
| **Health** | `GET https://mcp.modelclone.app/mcp/health` (no auth) |
| **Spec** | [Model Context Protocol](https://modelcontextprotocol.io) · [Claude MCP connectors](https://docs.claude.ai/en/docs/build-with-claude/mcp) |

## Session lifecycle

1. First `POST /mcp` with valid `mcl_` key → server creates `StreamableHTTPServerTransport` + in-memory session.
2. Response includes **`mcp-session-id`** header (UUID).
3. Follow-up requests send `mcp-session-id` to reuse the transport; **`entry.at`** refreshed on each hit.
4. Sessions expire after **30 minutes** idle (`SESSION_TTL_MS`); pruned every 5 minutes.

### Session binding

Each session stores the **`apiKey`** that initialized it. A follow-up request with a **different** key but the same `mcp-session-id` is rejected:

- HTTP **403**
- JSON-RPC code **-32001**
- Message: **"Session bound to a different API key"**

Do not swap keys on a live connector without reconnecting.

### After idle expiry or cold start

| Event | Client action |
|-------|---------------|
| 30 min idle | Re-`initialize`; new `mcp-session-id` issued |
| Vercel cold start | In-memory session gone; same as above |
| Lambda scale-out | Session may not exist on new instance; retry initialize |

Account data (generations, models, credits) is **not** stored in the MCP session — only transport state.

## V1 sunset on `/mcp`

`mcpSunsetGuard` runs before auth when `api_v1_sunset` flag is on. Returns **410** with `API_V1_SUNSET` + `v2BaseUrl` (see §3). Health endpoint is **not** gated.

## Cold starts (Vercel)

Sessions live **in memory** on the lambda instance. After a cold start, clients must **re-initialize** (new `initialize` / new session). Claude connectors usually handle this; custom clients should retry on session errors.

**Future improvement:** Redis-backed session store (not shipped).

## CORS

Allowed browser origins (production):

- `https://claude.ai` and `*.claude.ai`
- `*.anthropic.com`
- `localhost` / `127.0.0.1` (dev)

Other origins are rejected in production (`Origin not allowed for MCP`).

## Allowed hosts (transport)

`StreamableHTTPServerTransport` validates Host:

- `mcp.modelclone.app`
- `modelclone.app`, `www.modelclone.app`
- `localhost`, `127.0.0.1`, `[::1]`

## Headers clients may send

| Header | Purpose |
|--------|---------|
| `Authorization` / `X-Api-Key` | `mcl_` key |
| `mcp-session-id` | Session continuity |
| `mcp-protocol-version` | Protocol negotiation |
| `Content-Type` | `application/json` |
| `Accept` | `application/json, text/event-stream` |

## OPTIONS

`OPTIONS /mcp` and `OPTIONS /mcp/health` return **204** for CORS preflight.

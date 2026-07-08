## Purpose

**`api_v1_request`** is the generic escape hatch. It proxies any **integrator-product** route on `/api/v1` with the same method, path, query, and JSON body as REST. Typed tools (e.g. `get_me`, `generate_recreate`) are preferred when available — they are shorter to call and self-documenting.

| | |
|---|---|
| **Tool name** | `api_v1_request` |
| **REST mapping** | Any allowed `METHOD /api/v1{path}` |
| **Description** | Call any integrator-product route 1:1. Allowed paths: resource `modelclone://v1/route-catalog`. Request/response shapes: `modelclone://v1/openapi`. |

## Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `method` | `"GET"` \| `"POST"` \| `"PUT"` \| `"PATCH"` \| `"DELETE"` | **Yes** | — | HTTP method |
| `path` | string | **Yes** | — | Path **without** `/api/v1` prefix, e.g. `/generations`, `/nsfw/generate` |
| `query` | object (string/number/boolean values) | No | omitted | Query string key-values |
| `body` | object | No | omitted | JSON body for POST/PUT/PATCH |

## Example MCP tool call

```json
{
  "name": "api_v1_request",
  "arguments": {
    "method": "GET",
    "path": "/me"
  }
}
```

## Example response (success)

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

## Example response (upstream error)

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

## Example response (allowlist rejection)

Thrown before fetch when the path is not on the integrator product surface:

```json
{
  "ok": false,
  "error": "Route not on integrator product surface: POST /admin/stats. Use resource modelclone://v1/route-catalog."
}
```

## Poll / completion flow

- **Sync routes** (e.g. `GET /me`, `GET /pricing/generation`, `POST /generate/enhance-prompt`): the response in `body` is final — no polling.
- **Async generation submits** (e.g. `POST /generate/recreate`, `POST /nsfw/generate`): read the generation id from `body.generation.id` (or `body.generationId` / `body.generationIds[]` depending on route), then poll with **`wait_for_generation`** or **`get_generation`**. See §07 Generations tools.

## Common errors

| HTTP / shape | When | What to do |
|---|---|---|
| **401** in `httpStatus` | Missing or invalid API key | Send `Authorization: Bearer mcl_…` or `X-Api-Key: mcl_…` on the MCP connection |
| **402** / **403** in `httpStatus`, `body.message` starts with `"Insufficient credits"` | Balance too low for a submit | Call `get_me` for `totalCredits`; call `get_pricing_generation` for costs |
| **404** in `httpStatus` | Resource id not found or belongs to another account | Verify the id; re-submit if expired |
| **429** in `httpStatus` | Rate limit exceeded | Back off and retry; see `docs/public-api/04-rate-limits.md` |
| `{ "ok": false, "error": "Route not on integrator product surface…" }` | Path not on allowlist (billing, admin, session-only) | Use only paths from `modelclone://v1/route-catalog` |

## Credit cost notes

This tool does not charge credits by itself — costs depend on the route you call. Before any generation submit, call **`get_pricing_generation`** for live prices and **`get_me`** to confirm balance.

## Allowlist

Before fetch, `assertIntegratorRouteAllowed` checks `docs/generated/V1_ROUTE_INVENTORY.json` (`integratorProduct: true` only). Admin paths, billing, and internal callbacks are **not** on the allowlist.

## More examples

### GET /generations (history)

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

### POST creator-studio image (async)

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

### POST with integrator webhook

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

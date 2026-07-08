## Claude Code (recommended — remote HTTP)

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

### Session notes

Claude Code manages `mcp-session-id` automatically. If you see **"Session bound to a different API key"**, remove and re-add the server with one key. Idle sessions expire after **30 minutes**.

---

## Cursor (remote HTTP — recommended)

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

## Cursor / Claude Desktop (local stdio)

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

## Claude.ai (web)

Not an IDE — use the custom connector (§9). Same URL and bearer token as Cursor HTTP above.

---

## Verify connectivity (all clients)

```bash
curl -sS https://mcp.modelclone.app/mcp/health
```

No auth required. Expect `"version": "2.0.0"` and `"toolGroups"`.

Test REST + key separately:

```bash
curl -sS -H "X-Api-Key: mcl_YOUR_KEY_HERE" https://modelclone.app/api/v1/me
```

---

## Tips for agents

- Start every session with **`get_me`** and **`get_pricing_generation`**.
- Read **`modelclone://v1/route-catalog`** when unsure which path to call via `api_v1_request`.
- After async submits, use **`wait_for_generation`** — do not spam POST submits (rate limits).
- Poll every **5–10 s**, not sub-second.
- One in-flight generation when testing to avoid `GENERATION_QUEUE_FULL`.

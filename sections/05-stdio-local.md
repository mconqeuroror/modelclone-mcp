## When to use stdio

| Use stdio | Use remote HTTP |
|-----------|-----------------|
| Cursor / Claude Desktop without HTTP MCP | Claude.ai cloud connector |
| Local dev without DNS | Production shared connector URL |
| Air-gapped or custom agents | No local Node process |

Both use **`createModelcloneMcpServer`** — identical tools and allowlist.

> **Recommended for most users:** remote HTTP at `https://mcp.modelclone.app/mcp` (see §9–§10). Stdio is optional for local dev.

## Install

```bash
cd integrations/mcp-modelclone
npm install
```

Requires **Node 18+**.

## Environment

```bash
export MODELCLONE_API_KEY=mcl_your_key_here
export MODELCLONE_BASE_URL=https://modelclone.app   # optional; default shown
npm start
```

The process speaks **JSON-RPC over stdin/stdout**. Logs/errors go to stderr.

## Claude Desktop (`claude_desktop_config.json`)

**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`  
**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`  
**Linux:** `~/.config/Claude/claude_desktop_config.json`

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

Use **absolute paths**. Restart Claude Desktop after edits.

### Claude Desktop — HTTP alternative

If your Claude Desktop build supports Streamable HTTP connectors, prefer:

- URL: `https://mcp.modelclone.app/mcp`
- Header: `Authorization: Bearer mcl_YOUR_KEY_HERE`

Same config as Claude.ai (§9) — no local Node process.

## Cursor (stdio)

**Settings → MCP → Add server** or edit `.cursor/mcp.json`:

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

Reload the Cursor window. Confirm tools: `get_me`, `api_v1_request`, …

## Cursor / Claude Code — remote HTTP (recommended)

See §10 for copy-paste HTTP configs (no local Node).

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `MODELCLONE_API_KEY must be set` | Export `mcl_` key before start |
| Server exits immediately | Run `npm install` in `integrations/mcp-modelclone`; use absolute path in `args` |
| Tools empty in client | Restart IDE; check stderr for import errors |
| 401 on tool calls | Key revoked or typo — mint new key in Settings |
| `410 API_V1_SUNSET` | V1 retired — migrate to `v2BaseUrl` from error (HTTP stdio still hits `/api/v1`) |

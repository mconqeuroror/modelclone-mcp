# ModelClone MCP

Connect any MCP-capable AI (Claude.ai, Claude Code, Cursor, Windsurf) to your [ModelClone](https://modelclone.app) account with **1:1 REST parity** — models, image/video generation, Creator Studio, NSFW pipelines, Flow Studio, gallery, and more.

```
Endpoint:  https://mcp.modelclone.app/mcp
Health:    GET https://mcp.modelclone.app/mcp/health
Transport: Streamable HTTP (production) · stdio (local dev)
Auth:      X-Api-Key: mcl_…  or  Authorization: Bearer mcl_…
Version:   2.0.0
```

Get an API key: [modelclone.app](https://modelclone.app) → Settings → API (Business plan).

---

## Quick setup (Cursor)

`.cursor/mcp.json`:

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

Reload the window. Smoke test: *"Call ModelClone get_me and show my credits."*

Use **`https://mcp.modelclone.app/mcp`** — not `modelclone.app/mcp`.

---

## Documentation

| Doc | Audience | Contents |
|-----|----------|----------|
| **[QUICKSTART.md](./QUICKSTART.md)** | Anyone connecting a client | Claude.ai, Claude Code, Cursor setup, sessions, troubleshooting |
| **[MODELCLONE_MCP.md](./MODELCLONE_MCP.md)** | Operators & agent authors | **Complete handbook** — architecture, transport, security, recipes |
| **[TOOLS-REFERENCE.md](./TOOLS-REFERENCE.md)** | Integrators | All **110** tools — parameters, REST mappings, examples |
| **[sections/](./sections/)** | Maintainers | Editable shards (01–15); merge with `node scripts/merge-mcp-doc.mjs` in monorepo |
| **[STDIO.md](./STDIO.md)** | Local dev | stdio transport for Cursor / Claude Desktop |

### Section index

| # | File | Topic |
|---|------|-------|
| 01 | [sections/01-overview.md](./sections/01-overview.md) | Architecture & parity scope |
| 02 | [sections/02-deployment.md](./sections/02-deployment.md) | DNS, Vercel, health |
| 03 | [sections/03-authentication.md](./sections/03-authentication.md) | API keys, sessions |
| 04 | [sections/04-streamable-http.md](./sections/04-streamable-http.md) | HTTP transport |
| 05 | [sections/05-stdio-local.md](./sections/05-stdio-local.md) | Local stdio |
| 06 | [sections/06-tool-api-v1-request.md](./sections/06-tool-api-v1-request.md) | Escape-hatch tool |
| 07 | [sections/07-tools-convenience.md](./sections/07-tools-convenience.md) | Typed tools (canonical schemas) |
| 08 | [sections/08-resources.md](./sections/08-resources.md) | MCP resources |
| 09 | [sections/09-claude-connector.md](./sections/09-claude-connector.md) | Claude.ai connector |
| 10 | [sections/10-ide-setup.md](./sections/10-ide-setup.md) | Cursor & Claude Desktop |
| 11 | [sections/11-security-allowlist.md](./sections/11-security-allowlist.md) | Route allowlist |
| 12 | [sections/12-troubleshooting.md](./sections/12-troubleshooting.md) | Common errors |
| 13 | [sections/13-recipes.md](./sections/13-recipes.md) | End-to-end workflows |
| 14 | [sections/14-limitations.md](./sections/14-limitations.md) | REST parity limits |
| 15 | [sections/15-appendix.md](./sections/15-appendix.md) | Appendix |

---

## Tool inventory (summary)

**110 tools** — typed REST proxies + `wait_for_generation` (server poll) + `api_v1_request` (escape hatch).

| Group | Examples |
|-------|----------|
| Meta | `get_me`, `get_pricing_generation`, `api_v1_request` |
| Models | `list_models`, `wizard_*`, `models_status` |
| SFW generate | `generate_recreate`, `generate_free`, `creator_studio_*`, `enhance_prompt` |
| NSFW | `nsfw_*`, `nsfw_video_*`, `sexting_*` |
| Video & motion | `generate_motion_video`, `generate_complete_recreation` |
| Flow Studio | `flows_*` (10 tools) |
| Gallery | `gallery_*`, `set_username` |

Full per-tool schemas: **[TOOLS-REFERENCE.md](./TOOLS-REFERENCE.md)**

### MCP resources

| URI | Content |
|-----|---------|
| `modelclone://v1/route-catalog` | Integrator route inventory |
| `modelclone://v1/openapi` | OpenAPI contract |
| `modelclone://v1/getting-started` | Sequencing guide |

---

## Example workflow

**Model + SFW image:**

`get_me` → wizard or `create_model` → `models_status` → `generate_recreate` → `wait_for_generation`

Full JSON walkthroughs: [sections/13-recipes.md](./sections/13-recipes.md)

---

## Related repos

| Repo | Purpose |
|------|---------|
| [modelclone-api-V1](https://github.com/mconqeuroror/modelclone-api-V1) | Full REST API documentation |
| [modelclone-skills](https://github.com/mconqeuroror/modelclone-skills) | Agent skills (`npx skills add mconqeuroror/modelclone-skills`) |
| [modelclone-cli](https://www.npmjs.com/package/modelclone-cli) | `npm install -g modelclone-cli` |

---

## License

MIT — documentation. ModelClone API usage subject to [modelclone.app](https://modelclone.app) terms.

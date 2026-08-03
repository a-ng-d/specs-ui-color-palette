# Contract — MCP server (`mcp-ui-color-palette`)

- **Status**: Implemented
- **Last updated**: 2026-07-27
- **Source**: `mcp-ui-color-palette/README.md` + inventory of tools actually exposed to the `claude-ui-color-palette` plugin

Cloudflare Worker (`McpAgent` + Durable Objects, Cloudflare Agents SDK). Streamable HTTP transport at `/mcp`. Thin proxy to `api-ui-color-palette` — automatically prefixes calls with `/v1`.

## Palette generation

| Tool | Equivalent API endpoint | Auth |
| --- | --- | --- |
| `get_palette` | `POST /v1/get-palette` | No |
| `get_color_system` | `POST /v1/get-color-system` | No |
| `create_color_harmony` | `POST /v1/create-color-harmony` | No |
| `extract_dominant_colors` | `POST /v1/extract-dominant-colors` | No |
| `generate_code` | `POST /v1/generate-code` | No |
| `generate_colors_from_prompt` | `POST /v1/generate-colors-from-prompts` | No |

### `generate_code` — input

`base`, `themes`, `format?`, `colorSpace?`. An exact mirror of the `POST /v1/generate-code` contract (see `api-endpoints.md`) — does **not** take a `paletteData` input.

## Published palettes

| Tool | Auth | Equivalent API endpoint |
| --- | --- | --- |
| `list_published_palettes` | No | `GET /v1/list-published-palettes` |
| `list_my_published_palettes` | Yes | `GET /v1/list-my-published-palettes` |
| `publish_palette` | Yes | `POST /v1/publish-palette` |
| `get_published_palette` | No | `GET /v1/get-published-palette/:id` |
| `share_published_palette` | Yes | `POST /v1/share-published-palette/:id` |
| `unshare_published_palette` | Yes | `POST /v1/unshare-published-palette/:id` |
| `update_published_palette` | Yes | `POST /v1/update-published-palette/:id` |
| `unpublish_palette` | Yes | `DELETE /v1/unpublish-palette/:id` |

Auth: OAuth 2.1, discovered via `/.well-known/oauth-authorization-server`, Bearer token automatically injected by the MCP client.

## ⚠️ Drift observed vs. the README

The `claude-ui-color-palette` plugin also exposes a **`preview_palette`** tool (visual palette preview), used by the `ui-color-palette-scale-palette` skill. This tool **does not appear** in the `mcp-ui-color-palette/README.md` table as of 2026-07-27. To verify: outdated README, or a tool recently added server-side without a doc update. To fix one way or the other before relying on it for new development.

## Client configuration

VS Code/Copilot and Claude Desktop connect over HTTP to `https://mcp-uicp.<subdomain>.workers.dev/mcp` (see the README for config snippets). The `claude-ui-color-palette` plugin connects the same way, through its own skills/agents wrapper.

## See also

- [REST API](api-endpoints.md) — the underlying routes each tool proxies
- [Color System](../01-domain-model/color-system.md) — `get_color_system` in detail
- [Architecture](../00-overview/architecture.md) §3 — how the MCP path compares to the design-tool-host path

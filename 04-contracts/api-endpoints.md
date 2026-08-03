# Contract — REST API (`api-ui-color-palette`)

- **Status**: Implemented
- **Last updated**: 2026-07-27
- **Source**: `api-ui-color-palette/README.md`

Cloudflare Worker, all endpoints versioned under `/v1`. Wraps `engine-ui-color-palette` + Supabase (published palettes) + Mistral AI (prompt generation) + passkey auth.

## Palette generation

| Method | Endpoint | Role |
| --- | --- | --- |
| POST | `/v1/get-palette` | Generates a complete palette from `base` + `themes` |
| POST | `/v1/get-color-system` | Resolves a Color System (`SystemData`) against a generated palette |
| POST | `/v1/create-color-harmony` | Generates harmonies (complementary, analogous, triadic, tetradic, compound, square) from a base color |
| POST | `/v1/extract-dominant-colors` | Extracts dominant colors from a JPEG/PNG image (URL, raw data, or multipart upload) |
| POST | `/v1/generate-code` | Generates code/tokens directly from `base` + `themes` |
| POST | `/v1/generate-colors-from-prompts` | Generates a palette from a natural-language description (Mistral AI) |

### `POST /v1/generate-code`

Request body — **`base`/`themes`, not `paletteData`**:

```json
{
  "base": { "...": "..." },
  "themes": [{ "...": "..." }],
  "format": "css",
  "colorSpace": "RGB"
}
```

`format`: `css | scss | less | tailwind-v3 | tailwind-v4 | swift-ui | ui-kit | compose | resources | csv | native-tokens | dtcg-tokens | style-dictionary-v3 | universal-json`

## Authentication

| Method | Endpoint | Role |
| --- | --- | --- |
| GET | `/v1/authenticate` | Starts a passkey authentication flow (SSE) |

## Published palettes

| Method | Endpoint | Auth | Role |
| --- | --- | --- | --- |
| GET | `/v1/list-published-palettes` | No | Lists public palettes (paginated, searchable) |
| GET | `/v1/list-my-published-palettes` | Yes | Lists the authenticated user's palettes |
| POST | `/v1/publish-palette` | Yes | Publishes a new palette |
| GET | `/v1/get-published-palette/:id` | No | Fetches a published palette by ID |
| POST | `/v1/share-published-palette/:id` | Yes | Makes a palette public |
| POST | `/v1/unshare-published-palette/:id` | Yes | Makes a palette private |
| POST | `/v1/update-published-palette/:id` | Yes | Updates a published palette |
| DELETE | `/v1/unpublish-palette/:id` | Yes | Permanently deletes a palette |

## Stack

Cloudflare Workers · `@yelbolt/engine-ui-color-palette` · Mistral AI · Supabase (Postgres) · `@jsquash/jpeg` / `@jsquash/png`

## Environment variables

`MISTRAL_API_KEY`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_PALETTES_TABLE`, `SUPABASE_PALETTES_VIEW`, `AUTH_WORKER_URL`, `AUTH_URL`

## Relationship with `mcp-ui-color-palette`

The MCP server is a thin 1:1 proxy of these routes — see [MCP tools](mcp-tools.md). Any contract change must happen here first, then be reflected on the MCP side.

## See also

- [MCP server](mcp-tools.md) — tool-to-endpoint mapping for the same routes
- [Palette](../01-domain-model/palette.md) — `base`/`themes`, the shared unit of exchange with `/get-palette`, `/generate-code`, `/get-color-system`
- [Color System](../01-domain-model/color-system.md) — the `/get-color-system` contract in detail
- [Modals](../02-features/modals.md) — the Publication modal drives the published-palette endpoints

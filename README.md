# @pipeworx/boe-legislacion

Spain's consolidated legislation — every Spanish statute and royal decree kept
up to date with every amendment folded in, plus the daily table of contents of
the state gazette, from the Agencia Estatal Boletín Oficial del Estado (BOE).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1851+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `boe_search_law(query, ambito?, limit?, offset?)` — search consolidated
  legislation by words in the title (e.g. "Código Civil", "ley de propiedad
  horizontal"). Returns matching laws with their BOE id, legal rank, issuing
  department, scope (national/regional), key dates, and the official BOE page
  URL for each. Pass the returned id to `boe_law`.
- `boe_law(id, article?)` — one consolidated law by its BOE id. Without
  `article`: metadata (rank, department, consolidation status, dates), subject
  tags, notes, and a table of contents of its articles/sections ("bloques") —
  capped at 200 entries and marked `truncated` for large codes. With
  `article` (a block id from that table of contents, e.g. `"a1"` for Artículo
  1): the actual text of that one article across every historical version it
  has had, each dated, with the current one flagged.
- `boe_daily_summary(date)` — table of contents of one day's BOE issue: every
  disposition and announcement published that day, grouped by section and
  issuing department, with each item's BOE id and title. BOE does not publish
  every day (no Sunday issue, some holidays); those dates come back
  `found: false`, never an empty list.

## Auth

Keyless. The upstream open-data API requires no registration, key, or
click-through terms — it is served to anyone with `Accept: application/json`.

## Data sources

- Agencia Estatal Boletín Oficial del Estado (BOE), Legislación Consolidada —
  the consolidated-text database of Spanish state and regional legislation,
  kept current with every amendment.
- Agencia Estatal Boletín Oficial del Estado (BOE), Sumario — the daily table
  of contents of the state gazette.

## Notes

- **The Accept header is honoured on 200 only.** On any error (400/404/500)
  the upstream ignores `Accept: application/json` entirely and always answers
  a fixed small XML envelope (status code + text, empty data) regardless of
  what was requested. Every call here reads the body as text first and only
  parses it as JSON when it looks like JSON, so an error message is never
  lost to a failed `res.json()`.
- **The empty-result shape is not uniform.** A zero-match search answers 200
  with `"data": ""` — an empty *string*, not `[]` or `{}` — while a populated
  list and a single-record lookup are both array-wrapped (`"data": [...]`,
  even for exactly one record). `boe_search_law` and `boe_law` normalise this
  and report `found: false` with plain wording on a genuine miss, never an
  empty array dressed as success.
- **Article/section text has no JSON form at all.** Only the table of
  contents (`texto/indice`) is available as JSON; the actual block text is
  XML-only. There is no XML parser in the Workers runtime, so `boe_law` with
  `article` extracts paragraphs with targeted regex over BOE's fixed
  one-block, one-or-more-versions, paragraph-per-version XML shape — reliable
  for real statute text, not a general-purpose XML parser.
- **Large codes are enormous.** The Código Civil's table of contents alone is
  2,444 blocks. `boe_law` without `article` caps what it inlines and reports
  `articles.total` / `articles.truncated` rather than ever returning the whole
  thing — use `article` to read one block at a time.
- The upstream does not report a total match count for search — `limit` /
  `offset` paginate, and `shown` tells you how many came back this call.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "boe-legislacion": {
      "url": "https://gateway.pipeworx.io/boe-legislacion/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/boe-legislacion/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1851+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/boe_search_law \
  -H 'Content-Type: application/json' \
  -d '{"query":"Código Civil","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/boe_search_law`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "boe-legislacion": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-boe-legislacion"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-boe-legislacion
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Boe Legislacion data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

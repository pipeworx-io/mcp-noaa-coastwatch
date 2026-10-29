# @pipeworx/noaa-coastwatch

NOAA CoastWatch's satellite and model ocean record — sea surface temperature,
chlorophyll-a, winds, currents and the full NDBC moored-buoy archive — read
live from CoastWatch's two ERDDAP servers.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `noaa_coastwatch_search_datasets(query, protocol?, node?, limit?)` — full-text search of the
  dataset catalogue. Start here: every other tool takes a `dataset_id` from these results.
- `noaa_coastwatch_dataset_info(dataset_id, node?)` — variables, units, axes, time coverage,
  licence. Call it before querying data; variable names are case-sensitive.
- `noaa_coastwatch_tabledap(dataset_id, variables?, constraints?, node?, limit?)` — rows from a
  tabledap dataset, e.g. hourly NDBC buoy observations back to 1970.
- `noaa_coastwatch_griddap_point(dataset_id, variable, latitude, longitude, time?, depth?, node?)` —
  one gridded value at one time/lat/lon, snapped to the nearest grid cell.

## Auth

Keyless. Both nodes serve anonymously. A User-Agent is sent on every request.

## Data sources

- <https://coastwatch.pfeg.noaa.gov/erddap> — `node: "westcoast"` (default). ERD/SWFSC: satellite
  SST, chlorophyll, winds, currents, and `cwwcNDBCMet`, the NDBC buoy archive.
- <https://coastwatch.noaa.gov/erddap> — `node: "national"`. Operational VIIRS and ABI products.

The two hold different datasets. A `dataset_id` from one does not resolve on the other, and the
error for that is "Currently unknown datasetID=…", which reads as a broken id rather than a
wrong node.

## Traps

Protocol-level traps live in `shared/src/erddap.ts` (shared with `ioos-erddap`) and are documented
in full at the top of that file. The two that bite hardest here:

- **ERDDAP reports "nothing found" as HTTP 404 with a plain-text body**, not as an empty table —
  for a search that matches nothing AND for a tabledap query whose constraints exclude every row.
  The body is not JSON, so a naive parse loses the message entirely. That message is worth keeping:
  it names the dataset's real range ("No data matches time>=2050-01-01 because the numeric
  variable's source min=1970-02-26T20:00:00Z, max=2026-09-17T23:35:00Z").
- **Variable names are case-sensitive and often UPPERCASE.** NDBC water temperature is `WTMP`, air
  temperature `ATMP`, wind speed `WSPD`. A lowercase request is a 400 `Unrecognized variable`,
  which reads as "this buoy does not report water temperature".

One more, local to this pack: **the two nodes do not agree on their search columns.** The West
Coast node returns 17 columns including "Accessible"; the national node returns 15 and omits it.
Anything that indexed `rows[6]` would read the title on one and the ISO-19115 link on the other —
a wrong answer, never an error. The shared client indexes by column name.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "noaa-coastwatch": {
      "url": "https://gateway.pipeworx.io/noaa-coastwatch/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/noaa-coastwatch/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/noaa_coastwatch_search_datasets \
  -H 'Content-Type: application/json' \
  -d '{"query":"sea surface temperature","protocol":"griddap","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/noaa_coastwatch_search_datasets`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-noaa-coastwatch` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/noaa-coastwatch/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "noaa-coastwatch": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-noaa-coastwatch"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Noaa Coastwatch data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

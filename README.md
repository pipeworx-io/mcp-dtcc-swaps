# @pipeworx/dtcc-swaps

DTCC's public, real-time swap price dissemination feed (pddata.dtcc.com) —
individual swap transactions and their pricing terms, disseminated live under
CFTC Part 43 for CFTC-regulated swaps (commodity, credit, equity, FX,
interest-rate), SEC-regulated security-based swaps (credit, equity,
interest-rate) and Canadian-regulator swaps (credit, equity, interest-rate),
plus the index of the underlying intraday/end-of-day bulk dissemination files.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `dtcc_swaps_recent(regulator, asset_class, limit?)` — the most-recently
  disseminated swap transactions for a regulator/asset-class combo (e.g.
  CFTC/IR), most-recent first, out of the feed's rolling ~100-row window. Each
  record carries notional, fixed/floating rate terms, execution/effective/
  expiration dates, product identifiers, and `actionType`.
- `dtcc_swaps_dissemination_files(regulator, asset_class, granularity?, limit?)`
  — the index of DTCC's public bulk dissemination files (5-minute "slice"
  files, or one-per-day "cumulative"/EOD files) for a regulator/asset-class,
  with each file's row count and public download URL. This lists file
  pointers; it does not download or unzip them.

## Auth

Keyless. No account, key, or click-through terms were found anywhere in the
reachable surface (see the spike write-up below) — the feed is a public,
unauthenticated JSON API on a public S3 bucket with directory listing enabled.

## Data sources

- `https://pddata.dtcc.com/ppd/api/general/regionname` /
  `.../bucketname` — resolves the live public S3 bucket name (`kgc0418-tdw-data-0`
  as of 2026-09-23; the pack calls the bucket directly, the same two values the
  dashboard SPA itself reads at load).
- `https://kgc0418-tdw-data-0.s3.amazonaws.com/dashboard/Ticker.json` — backs
  `dtcc_swaps_recent`. A flat map of `{REGULATOR}_{ASSETCLASS}` (live keys:
  `CA_CR`, `CA_EQ`, `CA_IR`, `CFTC_CO`, `CFTC_CR`, `CFTC_EQ`, `CFTC_FX`,
  `CFTC_IR`, `SEC_CR`, `SEC_EQ`, `SEC_IR`) to an array of the ~100
  most-recently-disseminated trade records for that bucket. Continuously
  updated; this pack always fetches it fresh, never caches/mirrors it.
- `https://kgc0418-tdw-data-0.s3.amazonaws.com/dashboard/Slice.json` (intraday)
  and `.../Cumulative.json` (end-of-day) — back
  `dtcc_swaps_dissemination_files`. Same `{REGULATOR}_{ASSETCLASS}` keying,
  each entry pointing at a downloadable `.zip` of CSV trade data for that
  window/day, with a row count.

### Traps for the next person

- **These endpoints are undiscoverable from the DTCC site's own HTML/JS
  source by grepping for hostnames** — the Angular SPA at `pddata.dtcc.com/ppd/`
  resolves the bucket name from its own `/ppd/api/general/*` calls at runtime,
  and reads the JSON files with a *relative* path built from that response.
  The bucket name (`kgc0418-tdw-data-0`) is not literally present anywhere in
  the shipped JS bundle. It was found by loading the page in a real browser
  and reading its network requests, not by grepping the bundle. If DTCC ever
  rotates the bucket, `regionname`/`bucketname` are the two calls to re-probe.
- **`regulator`/`asset_class` combos are not a fixed enum** — CA and SEC never
  carry `FX` or `CO`, only CFTC does. Both tools throw a clear error naming the
  live keys actually present in that call's response rather than trusting a
  hardcoded list, since DTCC could add/retire a combo without notice.
- **Corrections and cancellations are NOT in-place edits.** A correction,
  cancellation, modification or termination of an earlier disseminated trade
  shows up as its own NEW record in `Ticker.json`, with `actionType` set to
  `CORR` / `EROR` / `MODI` / `TERM` instead of `NEWT`. Treat the feed as an
  append-only event log. A caller who wants only "new trades" should filter
  `actionType === "NEWT"` client-side; this pack returns the raw event stream
  so that behavior isn't silently baked in.
- **`Ticker.json` is a rolling window (~100 rows per bucket), not history.**
  For anything beyond that, `dtcc_swaps_dissemination_files` points at the
  underlying zip files (5-min slices or EOD cumulative), but downloading and
  unzipping those is deliberately out of scope for this pack — it returns the
  file index (name, URL, row count, window) so a caller can fetch the ones
  they actually need.
- **No licence/terms page was found.** `www.dtcc.com/legal.php` (the general
  corporate site) 403s on every fetch attempt (Akamai bot wall), unrelated to
  the `pddata.dtcc.com` data feed itself, which is unauthenticated throughout.
  The basis for proxying this data is that DTCC operates as a CFTC/SEC-
  registered Swap Data Repository and Part 43 of the CFTC's regulations
  mandates real-time PUBLIC reporting of swap transaction and pricing data —
  confirmed live via `www.cftc.gov/IndustryOversight/DataRepositories/index.htm`.
  Full probe trail: `docs/api-wave-2026-09-23-spike.md` (source #3).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dtcc-swaps": {
      "url": "https://gateway.pipeworx.io/dtcc-swaps/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dtcc-swaps/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/dtcc_swaps_recent \
  -H 'Content-Type: application/json' \
  -d '{"regulator":"CFTC","asset_class":"IR","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/dtcc_swaps_recent`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "dtcc-swaps": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dtcc-swaps"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-dtcc-swaps
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dtcc Swaps data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

# @pipeworx/ftc-enforcement

FTC (Federal Trade Commission) cases and proceedings — competition/merger enforcement and consumer-protection enforcement together, by company, case/matter number, topic, or date. Complements `hsr-notices` (the earliest signal that a merger was filed for antitrust review) and `enforcement-actions` (DOJ press releases and SEC litigation/administrative feeds) with the FTC's own case record: complaints, consent orders, settlements, case status, and the federal court involved.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1704+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ftc_search_cases(query?, category?, status?, start_date?, end_date?, limit=20)` — search cases/proceedings by company name, individual name, case/matter number, or keyword. `category` restricts to `merger`, `nonmerger` (conduct enforcement), or both (`any`, default). `status` restricts to `pending`, `closed`, or `under_order`.
- `ftc_case_detail(slug)` — one case's status, federal court (if any), full party caption, case summary, linked press release, and complete document timeline (complaint, orders, settlements, briefs) each with its filing date, title, and PDF URL. Takes the `slug` field returned by `ftc_search_cases`/`ftc_recent_actions` (the last path segment of the case's ftc.gov URL) or a full case URL.
- `ftc_recent_actions(days=30, category?, limit=20)` — the newest cases/proceedings, optionally restricted to merger or non-merger enforcement.

## Merger vs. non-merger

`category: "merger"` / `"nonmerger"` maps to the FTC's own "Competition Topics" facet (`field_competition_topics=708` / `711` on the live search form) — the only filter on ftc.gov's own search that actually splits merger enforcement from non-merger (conduct) enforcement. `field_case_action_type` (Federal/Administrative/ProcessEnforcement) and `field_enforcement_type` do not make that distinction.

## Data source

Scraped live from `https://www.ftc.gov/legal-library/browse/cases-proceedings`, the FTC's own Drupal Views search page. There is no structured feed for this content: `api.ftc.gov` (the JSON:API broker the `hsr-notices` pack uses) only documents two resource types — HSR early-termination notices and Do Not Call complaints (verified against [FederalTradeCommission/ftc-api-docs](https://github.com/FederalTradeCommission/ftc-api-docs), 2026-10-07) — and `/jsonapi`, `/v0/cases`, `/v0/legal-library-items` and similar guesses all 404. This pack issues the same GET a browser issues against the public search form and parses the HTML it returns: a live per-request proxy of public FTC data, not a mirrored dataset.

Every response carries `data_as_of` via each case's own `last_updated` field (the FTC's own "Last Updated" date) and `url`/`documents[].url` back to ftc.gov and the underlying PDFs.

## Reachability note

A plain GET with a descriptive User-Agent succeeds from a non-Worker vantage (verified 2026-10-07). `www.ftc.gov` runs some bot filter — a UA-less request even to `/robots.txt` gets an Akamai-style "resembles an abusive automated request" block page, while a UA'd request does not — which matches the ".gov UA-block" class this fleet has hit before, not necessarily the Cloudflare-egress-IP-block class that defeats a good UA outright (`loc.gov`, Kalshi, Overpass; see `mcps/chronicling-america`). That class can only be confirmed from the deployed Worker's own egress. `pwFetch` in `src/index.ts` is a direct fetch with a descriptive UA, and also accepts `_proxyUrl`/`_proxyToken` in the same shape `chronicling-america` uses — if a post-deploy probe shows 403s, the fix is: add `www.ftc.gov` to `supabase/functions/egress-proxy`'s `ALLOWED_HOSTS`, deploy that function, add `entry.slug === 'ftc-enforcement'` to the gateway's `EGRESS_PROXY_URL`/`EGRESS_PROXY_TOKEN` injection block in `workers/gateway/src/index.ts`, and redeploy the gateway. No pack code change needed.

## Markup contract (why the parsing looks the way it does)

- `items_per_page` must be one of `20`, `50`, `100` (the form's own `<select>` options) — any other value silently renders zero rows, not an error. This pack always requests the smallest valid page size that covers the caller's `limit` and slices client-side.
- Field extraction is scoped to the specific HTML block for the row/case/timeline-item in question, never the whole page — ftc.gov renders sidebar and mega-menu blocks (e.g. "Latest Press Releases") earlier in the document than the case's own content, using the exact same CSS class names for unrelated nodes. An unscoped scan returns a wrong-but-plausible value (e.g. today's date from an unrelated press release) rather than an error.
- A case's "Type of Action" (Federal/Administrative/ProcessEnforcement) and "Case Status" appear reliably in search-result rows; the case detail page carries case status, last-updated date, long caption, and federal court (federal cases only).
- Document dates come only from the case's "Case Timeline" entries; a single timeline entry can carry more than one PDF.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ftc-enforcement": {
      "url": "https://gateway.pipeworx.io/ftc-enforcement/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ftc-enforcement/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1704+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ftc_search_cases \
  -H 'Content-Type: application/json' \
  -d '{"query":"Kroger"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ftc_search_cases`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ftc-enforcement": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ftc-enforcement"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ftc-enforcement
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ftc Enforcement data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

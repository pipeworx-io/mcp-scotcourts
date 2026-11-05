# @pipeworx/scotcourts

Scottish court and tribunal judgment search — Court of Session, High Court of
Justiciary, the National Personal Injury Court, Sheriff Appeal Courts, Sheriff
Court civil/criminal, and the Scottish Upper Tribunal chambers, from the
Scottish Courts and Tribunals Service (SCTS), published from 1999 onward.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1866+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `search_scottish_judgments(query?, court?, from_date?, to_date?, page?, limit?)`
  — searches the judgment index; returns title, court, judges, date and a
  `document_link` to the PDF, plus the true corpus match count.
- `get_scottish_judgment(document_link?, query?)` — resolves a judgment
  `document_link` (or looks one up by case name/query first) and confirms the
  PDF is actually reachable: HTTP status, content-type, size. **Does not
  extract PDF text** — SCTS judgments are PDF-only and no PDF-text-extraction
  dependency exists anywhere in this catalog; `text_extracted: false` is
  always returned alongside `pdf_url`.

## Auth

Keyless. No API key, no cookie, no bearer token — verified live with a plain
`curl` (no browser session) against `api.pa.web.scotcourts.gov.uk/web/search`.

## Data sources

- `https://api.pa.web.scotcourts.gov.uk/web/search` (POST, JSON) — the
  judgment search index behind the client-rendered SPA at
  `scotcourts.gov.uk/judgments/#/`. **Do not use**
  `scotcourts.gov.uk/search-results/?filterType=Judgments` — that is the
  generic sitewide search (server-rendered HTML) and returns news articles and
  hearing/livestream pages mixed in with real judgments; it was ruled out as a
  dead end for this pack (fleet #2259).
- `https://www.scotcourts.gov.uk<documentLink>` — the judgment PDF itself.
  `documentLink` in the API response is relative; resolve it against this
  host.

### Traps found while building this (fleet #2259 spike, #2262 build)

- The judgments SPA and the sitewide search are two entirely separate apps on
  two different hosts. The static bundle referenced from `/judgments/` itself
  (`/apps/scts-web-apps.min.js`) is shared site-wide chrome and does not
  contain the judgments app's own logic — it is loaded dynamically at
  runtime, so the request shape was captured by driving the live page with
  devtools/a monkey-patched `window.fetch`, not by reading a static bundle.
- `filters` shape (proven live, both in-browser and via a bare `curl`):
  - Court: `{"field":"Court","value":"<exact dropdown string>"}` — values are
    NOT guessable from a court's common name (e.g. the dropdown says "Court of
    Session", not "Court of Session (Outer House)"). The full list is in
    `COURTS` in `src/index.ts`.
  - Date range: `{"field":"AdditionalDate","value":"<from>|<to>"}` — either
    side of the pipe may be blank for an open-ended range (`"2020-01-01|"` =
    from-only, `"|2020-01-01"` = to-only). Both forms verified live.
  - **An unrecognised `Court` value does not error** — it returns HTTP 200
    with `results: []` and `pagination.count.total: 0`, which reads exactly
    like "no such judgments" if not caught. This pack validates `court`
    against the known list before sending the filter.
  - The "Show more filters" panel behind the four basic fields (keyword,
    court, date-from, date-to) was **not** reverse-engineered — out of scope
    for this build; only what was proven ships.
- `pagination.count.total` is the true corpus match count; `results.length`
  is capped by the `limit` sent in the request. Surface `total`, never
  `results.length`, as "how many judgments matched."
- Coverage disclosure: this index is whatever SCTS has chosen to publish,
  from 1999 onward — not every Scottish judgment ever issued. Sheriff court
  and tribunal coverage is thin next to the higher courts. This is Scots law;
  England & Wales / UK Supreme Court / UK tribunal decisions are a different
  corpus (`@pipeworx/uk-caselaw`).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "scotcourts": {
      "url": "https://gateway.pipeworx.io/scotcourts/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/scotcourts/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1866+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/search_scottish_judgments \
  -H 'Content-Type: application/json' \
  -d '{"query":"negligence"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/search_scottish_judgments`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-scotcourts` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/scotcourts/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "scotcourts": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-scotcourts"]
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
ask_pipeworx({ question: "your question about Scotcourts data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

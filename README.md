# @pipeworx/kokkai-diet

Speech-level full-text search of Japanese National Diet (parliament) proceedings back to 1947 — both chambers, every recorded speech with speaker, party, session, date, and a permalink.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1573+ live data sources.

## Tools

- `search_speeches(query?, speaker?, speaker_group?, house?, from?, until?, start_record?, max_records?)` — full-text speech search by keyword and/or speaker, with optional party/house/date-range filters. Paginated (max 100/request).
- `search_meetings(query?, meeting_name?, house?, from?, until?, start_record?, max_records?)` — find meetings/sittings by name, chamber, or date range. Paginated (max 10/request — lower cap than speeches).
- `get_speech(speech_id)` — fetch one speech's full text and metadata by its speechID.

## Auth

Keyless.

## Data sources

- <https://kokkai.ndl.go.jp/api/speech> — speech-level full-text search.
- <https://kokkai.ndl.go.jp/api/meeting> — meeting-level search.

Both are JSON/XML endpoints of the National Diet Library's 国会会議録検索システム (Kokkai Kaigiroku Search System).

## Traps for the next person

- **The API requires a real search anchor.** `from`/`until`/`nameOfHouse` alone are accepted (200 OK) but silently return zero records — they're filters, not search conditions. `search_speeches` requires `query` and/or `speaker`; `search_meetings` requires `query`, `meeting_name`, or `house`. A no-condition request 400s with `(19007)検索条件を指定してください`.
- **Per-request record caps differ by endpoint, and out-of-range values 400 rather than clamp.** `/api/speech`: `maximumRecords` 1–100 (default 30). `/api/meeting`: `maximumRecords` 1–10 (default 3) — easy to miss since it's a third the speech cap. Both tools here clamp client-side to the valid range, but a caller reading raw upstream docs will hit `(19005)`/`(19006)` if they don't.
- **Never let a capped page read as the complete result.** Both tools return `next_start_record` (from the API's `nextRecordPosition`) and `is_complete_result` (`true` only when `next_start_record` is `null`). `total` (upstream `numberOfRecords`) is the true count — a caller must check it against `returned` before assuming they've seen everything.
- **Queries are Japanese-first.** The full-text index is Japanese; partial-match works on Japanese substrings. English keywords (e.g. "semiconductors") will under-match or return zero even when substantial Japanese-language debate exists on the topic — use Japanese keywords (e.g. "半導体"). Tool descriptions say this explicitly so an empty result isn't misread as "no debate on this".
- **`search_speeches` returns a speech preview (400 chars), not the full text** — fetch the complete speech with `get_speech(speech_id)` using the `speech_id` from the search hit. This keeps list responses from blowing up on speeches that run thousands of characters.
- **400 error bodies from the upstream carry a useful Japanese message + details array** (e.g. `(19005)maximumRecordsには1～100の値を指定してください`) — this pack surfaces that text verbatim rather than a generic HTTP error, since it tells the caller exactly which parameter is wrong.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "kokkai-diet": {
      "url": "https://gateway.pipeworx.io/kokkai-diet/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/kokkai-diet/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1573+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "kokkai-diet": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-kokkai-diet"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-kokkai-diet
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Kokkai Diet data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

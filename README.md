# @pipeworx/sigstore-rekor

The Sigstore Rekor public transparency log — the immutable, append-only record
of software signing events behind cosign, npm and PyPI provenance, and SLSA
attestations.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `rekor_log_info(...)` — current log state: total entries, signed tree head, Merkle root, retired shards. Call it for the upper bound on `log_index`.
- `rekor_entry_by_index(log_index)` — one entry by its integer position, body decoded.
- `rekor_search_by_hash(hash | email)` — given a sha256 artifact digest (or a signing-certificate email identity), every entry that signed it. Returns uuids.
- `rekor_entry(uuid)` — one entry by uuid, with signing certificate, integration time and inclusion proof.

## Auth

Keyless.

## Data sources

- <https://rekor.sigstore.dev/api/v1/log> — signed tree head and shard list.
- <https://rekor.sigstore.dev/api/v1/log/entries?logIndex=N> — entry by position.
- <https://rekor.sigstore.dev/api/v1/index/retrieve> — POST `{hash}` or `{email}`, returns uuids.
- <https://rekor.sigstore.dev/api/v1/log/entries/{uuid}> — entry by uuid.

Notes the next person would otherwise rediscover:

- `body` is base64-encoded JSON. Every tool here decodes it into `entry_body`
  and lifts the artifact digest and entry kind to the top level; a caller that
  reads the raw field gets an opaque blob.
- `logIndex` and `uuid` are different identifier spaces. `rekor_search_by_hash`
  returns uuids, so it feeds `rekor_entry`, not `rekor_entry_by_index`.
- Entry KINDS (`rekord`, `hashedrekord`, `intoto`, `dsse`) do not share a body
  shape. `summarize()` looks each field up where that kind puts it and returns
  null where the kind has no equivalent — do not assume `artifact_digest` is
  always populated.
- The index search hashes want the `sha256:<hex>` form. A bare hex digest is
  what every other tool prints, so both are accepted and normalised in-pack.
- No entry for a digest is NOT evidence the artifact is bad — only that there
  is no public signing record. The tool says so in its `note`.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sigstore-rekor": {
      "url": "https://gateway.pipeworx.io/sigstore-rekor/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sigstore-rekor/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/rekor_log_info \
  -H 'Content-Type: application/json' \
  -d '{}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/rekor_log_info`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "sigstore-rekor": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sigstore-rekor"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-sigstore-rekor
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Sigstore Rekor data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

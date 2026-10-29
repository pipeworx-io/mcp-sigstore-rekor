# @pipeworx/sigstore-rekor

The Sigstore Rekor public transparency log — the immutable, append-only record
of software signing events behind cosign, npm and PyPI provenance, and SLSA
attestations.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/rekor_log_info \
  -H 'Content-Type: application/json' \
  -d '{}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/rekor_log_info`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-sigstore-rekor` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/sigstore-rekor/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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
ask_pipeworx({ question: "your question about Sigstore Rekor data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

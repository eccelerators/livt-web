# Bounded HTTP request parsing

`HttpRequestParser<S, MAX_BYTES = 1024, MAX_TARGET = 128, MAX_HEADERS = 16>`
implements `IHttpRequest` in `Livt.Web.Http`. It borrows one `IPacketData` source
and validates one complete bodyless HTTP request before publishing read-only
views. It owns scanner state and bounded header descriptors, not request bytes.

```livt
parser = new HttpRequestParser<RequestData, 1024, 128, 16>(requestData)
// After the application publishes its stable, initialized request bytes:
var result = parser.TryParse()
if (result == HttpRequestResult.Ready) {
    var value: byte = 0x00
    var readResult = parser.TryReadPath(0, value)
    // Use value only when readResult == PacketDataResult.Success.
}
// Finish all consumers before invalidating and reusing requestData.
parser.Invalidate()
```

The type `RequestData` above denotes an application-provided component type.
The existing demo server still uses its original recognizers; its migration to
this parser and the new router is a later integration step.

## Supported input

- Case-sensitive token methods, origin-form targets beginning with `/`, HTTP/1.0
  or HTTP/1.1. Get/Head/Other are classifications, not implemented behaviors.
  A bodyless POST parses as Other, never Get. Routing decides whether a method
  is allowed; this permits a future 405 response for a known path. GET remains
  the initial application's supported handler method.
- Strict single spaces between request-line elements and CRLF line endings.
- Complete headers ending with an empty CRLF line; no obsolete folded fields.
- HTTP/1.1 requires exactly one nonempty Host field. HTTP/1.0 may omit Host.
- The current Host subset is DNS-style labels/dotted IPv4 text, optionally followed
  by a decimal port in 0..65535. IPv6 literals and other authority forms are
  unsupported. Host is not compared with the configured listening address.
- No request body: Content-Length may be absent or consist only of zero digits.
  Leading zeroes are accepted. Nonzero lengths are Unsupported, without integer
  accumulation or overflow. Duplicate Content-Length fields (even identical),
  signed/list/empty lengths, and Content-Length plus Transfer-Encoding are Malformed.
- Transfer-Encoding and Expect are Unsupported. Empty or repeated instances are
  Malformed. Unrecognized headers are retained as checked views, not interpreted.
- Extra bytes after the header terminator are Unsupported, including a second
  pipelined request. The caller must provide an HTTP request extent, not a padded
  Ethernet frame. This parser neither reassembles TCP nor handles connection state.

These are intentional bounded-subset policies. Strict line/field parsing and
framing handling follow the structure described in
[RFC 9112](https://www.rfc-editor.org/rfc/rfc9112.html#section-2.1); accepting this
subset is not a claim of full HTTP/1.1 server conformance.

## Paths, queries and headers

Path/query views retain encoded bytes. Split the target at its first literal `?`;
subsequent `?` bytes belong to the query. HasQuery distinguishes no query from an
empty query. Percent escapes must contain two hex digits but are not decoded.
There is no case folding, dot-segment removal, slash collapsing or trailing-slash
normalization. `/a`, `/%61` and `/a/` remain distinct. Raw fragments, spaces,
controls, non-ASCII target bytes and invalid URI characters are rejected. Target
character rules use the path/query structure in
[RFC 3986](https://www.rfc-editor.org/rfc/rfc3986.html#section-3.3).

Header names are validated as tokens and preserved in original case. GetHeaderKind
recognizes Host, ContentLength, TransferEncoding and Expect case-insensitively;
other fields return Other. Indexed name/value reads allow applications to inspect
other headers without copying them. Leading/trailing SP and HTAB are trimmed from
value views; interior whitespace and opaque high-bit value bytes are preserved.
Unknown repeated fields remain separate entries, in received order. No implicit
merging or comma-list interpretation occurs.

| Request view | Operations |
|---|---|
| Publication | IsReady |
| Request line | GetMethod, GetVersion |
| Path | GetPathLength, TryReadPath |
| Query | HasQuery, GetQueryLength, TryReadQuery |
| Headers | GetHeaderCount, GetHeaderKind, GetHeaderNameLength, GetHeaderValueLength |
| Header bytes | TryReadHeaderName, TryReadHeaderValue |

Header indexes are zero-based. Invalid metadata indexes return zero/Other;
invalid byte indexes return Invalid. When unavailable, metadata returns neutral
values and byte reads return NotReady. There are no raw-array public methods.

## Results and limits

| Result | Meaning |
|---|---|
| NotReady | Source is not published, or the parser has been invalidated |
| Ready | One complete structurally supported request has been published; routing still checks its method |
| Incomplete | Available bytes end before a complete required construct |
| Malformed | Invalid syntax or rejected ambiguous framing |
| Unsupported | Recognized input outside the supported request subset |
| TooLarge | Request bytes, complete target including query, or field count exceeds its bound |
| ReadFailure | A provider read fails or advertises a negative extent |

All three capacities are positive compile-time parameters. A source exceeding
MAX_BYTES is rejected before any byte read. MAX_HEADERS counts every field,
including unknown fields, but not the terminating blank line. Field byte lengths
are bounded by MAX_BYTES rather than separate copied name/value arrays. Header
descriptor memory grows with MAX_HEADERS; route count does not duplicate it.

Results describe the first detected failure; the parser does not continue after
failure to enumerate all defects. An unsupported request is never published.
Incomplete is not an incremental parser state: invalidate/release the old borrow,
publish a new assembled extent, and parse again. No success is inferred from a
GET prefix or from bytes outside the supplied provider's initialized bounds.

## Lifetime and failures

One serialized owner calls the graph in one clock/reset domain. Hold the source
published and unchanged throughout parsing and all dependent request reads.
TryParse starts by invalidating the prior view; do not call it while a handler
still borrows that view. End consumers before Invalidate, then release/reuse the
source. Invalidate does not clear or release source storage. Source republication
must be preceded by parser invalidation, even when the new extent has the same
length; providers do not expose generation identifiers.

After successful parsing, a failed view read invalidates publication with
ReadFailure and propagates the provider's PacketDataResult. Failed output bytes
are zeroed. Bounds errors are Invalid and leave an otherwise valid publication
intact. An unpublished source makes IsReady false. Reset all owners together;
parser metadata is not usable until a new successful parse.

## Verification

The main package test manifest includes HttpRequestParserTest and
HttpRequestBoundsTest. They exercise complete views and method classification, truncation at every offset
of a minimal request, malformed lines/fields, framing conflicts, unsupported
requests, exact/over-limit capacities, provider failures and storage reuse.
Only Livt tests are required; no synthesis or board test is implied.

### Recorded result (2026-09-24)

The two new test components pass **51 tests, 0 failed, 0 skipped** with default
optimizations in an isolated project using the actual sibling Livt.Net checkout
(commit `63fe5a84a54c62b77425e5da5feab93a3dd57556`). Command:
`env -u _JAVA_OPTIONS livt test` in `/tmp/web002/focused`.
Both component simulations complete (capacity: 78,225 ns; parser: 1,454,285 ns).
Evidence: `/tmp/web002/focused-tests.log` and `promoted-sources.json`.
Promoted sources match the tested code; the corpus gained explanatory comments.
The real package's `livt test --list` discovers all 90 tests, including these 51.
The existing 39 server tests were not rerun: their implementation and consumers
are unchanged. Discovery is not reported as execution evidence.

Source-level compiler workarounds are documented in `../COMPILER.md`. The strict
capacity and malformed-input expectations remain enabled. No synthesis or board
verification was performed for this parser addition.

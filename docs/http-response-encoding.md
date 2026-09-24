# Bounded HTTP response encoding

`HttpResponse<P>` and `HttpResponseEncoder<R>` in `Livt.Web.Http` expose a
prepared HTTP/1.0 response through Livt.Net's checked `IPacketData` contract.
They are independent of Ethernet, TCP, AXI and storage implementations. The
existing demo server still uses its original fixed response components until
the application/transport migration.

## Composition

```livt
body: ArrayPacketData<256>
response: HttpResponse<ArrayPacketData<256>>
encoder: HttpResponseEncoder<HttpResponse<ArrayPacketData<256>>>

new()
{
	body = new ArrayPacketData<256>()
	response = new HttpResponse<ArrayPacketData<256>>(body)
	encoder = new HttpResponseEncoder<HttpResponse<ArrayPacketData<256>>>(response)
}
```

After writing and publishing the body, call:

```livt
var described = response.TryPrepare(HttpStatus.Ok, HttpContentType.Html, HttpAllowedMethods.None)

if (described) {
	var prepared = encoder.TryPrepare(responseByteBudget)
	// Only a successful preparation may be offered to the transport.
}
```

The budget is the complete HTTP message size, including headers. The adapter
chooses it according to its own transport/storage limits. There is no default
MTU or frame offset in these components. `GetHeaderLength()` exposes the prepared
HTTP body offset; `GetAvailableLength()` exposes the complete HTTP byte count.
`TryRead(index, value)` reads the actual published body after the headers.
The resulting encoder can be used wherever Net accepts an `IPacketData` payload;
checksums are computed by Net over that stream, never supplied by the caller.

A handler/router can implement `IHttpResponse` directly instead of using
`HttpResponse<P>`. The interface extends `IPacketData` with `GetStatus()`,
`GetContentType()` and `GetAllowedMethods()`. Its bytes are **body bytes only**.
The encoder independently validates metadata and length from any implementation.
Use one shared encoder over the selected response, not one per route.

## Supported wire format

| Metadata | Values |
|---|---|
| HttpStatus | Ok (200), BadRequest (400), NotFound (404), MethodNotAllowed (405), InternalServerError (500) |
| HttpContentType | Html (`text/html`), PlainText (`text/plain`), Json (`application/json`) |
| HttpAllowedMethods | None, Get, Head, GetAndHead |

`Unknown` status/content type is rejected. A 405 requires a nonempty allowed-method
set; other supported statuses require None. `Allow` is emitted only for 405,
with `GET`, `HEAD`, or `GET, HEAD`. This follows the [405 response requirement](https://www.rfc-editor.org/rfc/rfc9110.html#name-405-method-not-allowed).
Advertising HEAD is metadata support, not an implementation of HEAD response
semantics. The application must advertise only methods it implements. The
initial application remains GET-only. This encoder always emits its body;
it is not suitable for HEAD, 1xx, 204 or 304 responses without a future extension.

Header order is status line, Content-Type, optional Allow, Content-Length,
Connection: close, then the empty line. Every line uses CRLF. There is no Date
header because the core has no wall clock, and no arbitrary user-supplied header
text. Bodies are opaque bytes: there is no JSON serialization, character encoding,
HTML generation, content lookup or body validation in the framework.

## Preparation, bounds and lifetime

- Publish the body before preparing its response. Metadata, length and bytes must
  remain stable until all encoder and transport readers finish, including any
  retransmission retention. Calls in the shared graph must be serialized.
- Preparation snapshots metadata and derives Content-Length from the published
  body length. It caches reversed decimal digits once, including the single `0`
  digit for an empty body. Header reads perform no decimal division and no body
  reads. The encoder owns ten digit bytes and scalar descriptors, not a header
  buffer or a duplicate body store.
- Negative body lengths/budgets, unsupported metadata and oversized responses fail
  without publication. The total-size check subtracts the body from the budget
  before comparing the header, avoiding signed integer addition overflow.
  Nonnegative 32-bit lengths are supported if the entire message fits the budget.
- Failed preparation and explicit Invalidate leave length zero and reads NotReady.
  Out-of-range reads return Invalid with a zero output and preserve publication.
  Body read failures return the original PacketDataResult, clear publication,
  and leave output zero even if the provider wrote a nonzero failure value.
  Consumers must still check the result; zero is not a replacement body byte.
- Preparation validates descriptors, not every body byte. A failed read during
  checksum preparation must prevent packet publication. A failed read after
  transmission begins requires the transport to abort; bytes already transmitted
  cannot be recalled. Never append a replacement error response mid-transfer.
- Invalidate the transport's dependent views, then encoder, then response, then
  release/reuse the body. Neither component releases storage it borrows.
  Observed source unpublication latches invalidation; it cannot silently restore
  readiness. Providers carry no generation token, so releasing and republishing
  storage behind live readers is prohibited even if those readers did not observe
  the unpublished interval.

## Verification

`HttpResponseEncoderTest` covers independent literal wire headers and full body
comparisons for all 15 status/media combinations, the three allowed-method sets,
empty/odd/even bodies, decimal boundaries, exact/insufficient budgets, integer
overflow, invalid metadata, source lifetime, checked bounds, failure propagation
and re-preparation. Large synthetic lengths test headers and endpoint bytes
without allocating or scanning gigabytes. Test providers deliberately write
nonzero bytes on failure to check output clearing.

Run with `env -u _JAVA_OPTIONS livt test -r HttpResponseEncoderTest`. Verification
uses Livt tests only; no synthesis or hardware resource/timing claim is implied.

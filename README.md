# Livt.Web

`Livt.Web` provides reusable HTTP/1.0 application-layer components for Livt
hardware designs. It sits above `Livt.Net` and builds compact, deterministic
web endpoints for FPGA projects that exchange complete Ethernet frames.

The package intentionally supports a narrow embedded-web subset: HTTP GET
recognition, fixed-body HTTP/1.0 responses, a minimal single-connection TCP
handshake, and web endpoint dispatch over `Livt.Net` ARP, ICMP, IPv4, and TCP
helpers.

## Package

```toml
[dependencies]
Livt.Web = "0.1.0"
```

`Livt.Web` depends on `Livt.Net 1.1.0-dev` for Ethernet, ARP, ICMP, IPv4, TCP,
checksum, and frame-composition primitives.

## Namespaces

Production components live in `Livt.Web.Http`. Tests use
`Livt.Web.Tests.Http`.

| Area | Components |
|---|---|
| HTTP recognition | `HttpGetRootRecognizer`, `HttpGetRecognizer`, `HttpRequestRecognizer` |
| HTTP response | `HttpResponseGenerator`, `HttpResponseFrameComposer` |
| Server state | `HttpServer` |
| Endpoint facade | `NetworkEndpoint`, `WebServer` |

## API Overview

### Recognizers

`HttpGetRootRecognizer` detects the exact token `GET / ` at a configured frame
offset. `HttpGetRecognizer` generalizes that shape with a configured path of up
to seven bytes. `HttpRequestRecognizer<S>` matches a configured token in a bounded HTTP payload.
`HttpServer` borrows Ethernet/IPv4 parsing, owns its TCP descendant and applies local endpoint/flag policy.

### Response Generation

`HttpResponseGenerator` emits an `HTTP/1.0 200 OK` response with
`Content-Type: text/html` and a decimal `Content-Length`. The caller supplies
a published `IPacketData` body to `HttpResponseFrameComposer<P>`. It prepares
`HttpResponsePayload<P>` inside Net TCP/IPv4/Ethernet components. TCP checksums
cover the actual HTTP header/body byte stream, including odd header lengths.

### Server And Endpoint Flow

`NetworkEndpoint<C>` and `WebServer<C>` take `content` as the first constructor
argument, followed by local MAC, IP and port. The endpoint owns one receive RAM
and one Ethernet/IPv4 parser pair shared by Net services and HTTP.

For custom composition, `HttpServer<S: IPacketData, C: IHttpContent>` takes
`(ethernet, ipv4, content, localMac, localIp, localPort)`. It borrows those parsers
and the published capture behind them; it does not allocate, load, publish or
release receive storage. Its owner closes request views before source reuse.
Use `InvalidateRequest()` before lending shared parsers to another consumer;
`BeginFrame()` also clears responses and releases selected content while retaining
connection state. Paths remain configurable:

```livt
SetRoutePath(route, path)
AcceptedRoute(route)
```

`NetworkEndpoint` delegates ARP/ICMP preparation and Ethernet/IPv4 classification
to `Livt.Net.NetworkService`, while `HttpServer` retains HTTP routing and the
existing TCP connection state. `WebServer<C>` is a compact facade over
`NetworkEndpoint` for applications that want a single web-facing component.
`IHttpContent.TrySelect(route)` publishes a stable body and `Release()` ends that
publication. The endpoint is the exclusive selector/releaser of its bound content.
The content owner must not modify the selected bytes during preparation or
emission. Finish all response reads before `BeginFrame()` releases the previous response;
repeated `HandleFrame()` calls preserve an already selected response.

The common loaded-frame flow is:

1. `BeginFrame()`
2. `LoadRxByte(index, value)` for each captured byte, appending in ascending order
3. `HandleFrame()`
4. `HasResponse()`
5. `GetResponseLength()` and `TryReadResponse(index, value)`; use `value` only on Success

Reception uses a published bounded byte provider. `BeginFrame()` invalidates prior
parser views before releasing storage; only bytes loaded for the current frame
are readable. IPv4 total length limits TCP payload recognition, so Ethernet
padding cannot supply missing request bytes. Supported header-only captures may
match a complete `GET <path> ` token; this is not complete HTTP/TCP validation.
Receive checksums remain unchecked. ICMP replies require a complete IP capture.

`HttpRequestRecognizer<S: IPacketData>` binds an HTTP payload provider and exposes
`IsGetRequest()` plus path configuration. It no longer accepts a raw frame, IP
address or port. `HttpServer` parses its Net protocol graph once per frame and
applies endpoint/flag policy before invoking its route recognizers.

## Scope

In scope:

- HTTP/1.0 GET recognition over fixed-header Ethernet/IPv4/TCP frames
- Route slots for short configured paths
- HTTP/1.0 200 OK response frames with caller-owned body bytes
- Minimal TCP SYN, ACK, PSH+ACK flow for one connection at a time
- ARP and ICMP endpoint dispatch through `Livt.Net`

Out of scope: IPv6, UDP application protocols, TLS, QUIC, TCP option negotiation, SACK,
window scaling, congestion control, chunked encoding, compression, keep-alive,
POST handling, dynamic content stores, and multiple simultaneous connections.

## Build And Test

```sh
livt test
```

The configured test list is defined in [`livt.toml`](livt.toml). To force test
regeneration without deleting synchronized dependencies:

```sh
livt test -f
```

## Development Notes

- Keep Ethernet, ARP, IPv4, ICMP, TCP, checksum, and frame-I/O primitives in
  `Livt.Net`.
- Keep HTTP recognition, response generation, TCP/HTTP server state, and web
  endpoint dispatch in `Livt.Web.Http`.
- Keep route meaning, page content, and board integration in application
  packages.
- Keep receive offsets and bounds inside protocol components; pass bounded
  payload providers to HTTP recognition. Prepare response graphs separately from
  receive parsing, then propagate checked read failures.
- Document compiler workarounds only while they remain reproducible.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).

Development builds use the sibling `livt-net` checkout with Livt.IO 1.2.0-dev.
Fixtures publish an initialized capture before parsing and invalidate views before
storage reuse. Livt.Web has no AXI fixtures; link admission, completion and
backpressure belong to Livt.Net and the application capability tests.

The sibling dependency is required for this development API: an existing registry
release of Web 0.1.0 is not evidence of compatibility with the redesigned Net.
Keep local dependencies during development; select new compatible release versions
and refresh locks together before publishing Web and its consumers.

Network checksums and header encoders use Livt.Net's static helper API. The
composers supply explicit address/port arrays and keep HTTP state in this package;
no checksum or byte-builder component instances are required.

The former `SetBodyConfig`/per-byte `GetResponseByte` facade API is removed.
Implement `IHttpContent` to adapt static, RAM-backed or application-generated
bodies; the selected provider determines length and bytes. There is no externally
supplied TCP checksum. Failed preparation exposes no response. See WebApp's
`WebContent` for a concrete store adapter and the full-frame tests for minimal
provider-bound construction. HTTP bodies must fit the standard Ethernet frame.

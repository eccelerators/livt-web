# Livt.Web

A bounded HTTP application layer for FPGA designs. Routing, request parsing and
response encoding are independent of AXI, receive RAM and the Ethernet driver.
Requires Livt.Net 1.1.0-dev; development builds resolve its sibling checkout.

## Composition

- `HttpRequestParser<S>` parses one complete bounded request and exposes checked
  path, query and header views over the borrowed source.
- `HttpPath<R, P>` compares an exact encoded URL path against application-owned
  immutable bytes. `Route<R, M, H>` binds a method, matcher and handler;
  `RouteChain<A, B>` composes ordered routes without per-route packet buffers.
- `StaticContent<P>` publishes an existing body. Implement `IHttpHandler` for
  pending work or application actions; Begin accepts once, Poll resumes and End
  closes the transaction. Keep published body bytes stable until End.
- `HttpRouter<T>` selects a handler or shared 404/405 response. `HttpApplication<S, T>`
  owns one encoder and coordinates request/handler/publication lifetimes.
- `HttpServer<S, T, CAPACITY = 1514>` is the default Ethernet assembly: it owns a
  router, application and `HttpNetworkEndpoint` around the supplied shared parser
  graph and route tree. It never owns or copies receive storage.

Construct the capture provider and Ethernet/IPv4/TCP parsers once, then construct
`HttpRequestParser<TcpConnectionRecognizer<S>>` over that same TCP view. Bind every
route and matcher to that request instance. Finally construct:

```livt
var server = new HttpServer<S, Routes>(
    source, ethernet, ipv4, connection, request, routes,
    localMac, localIp, localPort)
```

`S` and `Routes` above stand for the concrete provider and route-tree types.
`livt-web-app` supplies a complete consumer with application-owned `/`, `/about`
and `/status` declarations, static pages and a dynamic status snapshot.
There are no library-owned URL IDs or fixed route slots.

Advanced consumers can compose `HttpRouter`, `HttpApplication` and
`HttpNetworkEndpoint` separately, or use `IHttpApplication` with another transport.
See [application and transport contracts](docs/http-application.md),
[request parsing](docs/http-request-parser.md), [routing](docs/http-routing.md),
and [response encoding](docs/http-response-encoding.md).

## Ownership and completion

1. Acquire/publish a complete receive extent after the preceding `TryRelease`
   succeeds. The driver and parsers must share a reset domain.
2. Call `HandleFrame`, then `Poll` while Pending. Never reuse RX while active.
3. When Prepared, check the complete frame length and every `TryRead` result.
   Retain the request, handler and body through transmitter admission/backpressure.
4. After successful local emission call `MarkEmitted`, then `Complete` once all
   readers have stopped. On failure, stop readers before calling `Abort`.
5. Call `TryRelease` before releasing RX or changing any borrowed content.

A response-size budget failure publishes nothing. One encoder reads each selected
body; no application-supplied checksum or raw body-byte injection hook is needed.

## Supported subset

The core handles bounded bodyless requests, exact encoded paths and explicit method
routes, plus 404/405 with Allow metadata. It does not decode/normalize paths or
provide middleware, streaming bodies, TLS or general HTTP server behavior. HEAD
body suppression is not implemented; the demo declares GET routes only.

The included Ethernet adapter is the narrow single-peer demo TCP transport:
one complete request in one captured segment, one response fitting the 1460-byte
HTTP budget, FIN-bearing response and no retransmission, reassembly or reliable
TCP session engine. Receive checksum/window validation limitations are detailed
in the transport document. Complete header terminators are required; captured
request prefixes are rejected. The default parser bound is 1024 HTTP bytes.

## Verification

Run `livt test --events`. Tests compare independent HTTP bytes and complete network
frames and cover parsing bounds, routing priority, handler failures, retention,
cancellation and provider failures. Livt tests do not establish FPGA utilization
or timing. No synthesis is required for ordinary framework iteration.

The legacy `WebServer`, `NetworkEndpoint`, numeric route setters, GET recognizers,
`IHttpContent` and old response generators/composers have been removed. `HttpServer`
now denotes the composable assembly above; consumers must migrate to its new API.

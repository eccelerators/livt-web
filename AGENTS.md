# AGENTS.md — Livt.Web

## Project Goal

Provide reusable HTTP/1.0 application-layer components for FPGA projects in the
`Livt.Web.Http` namespace. The package builds on `Livt.Net` for Ethernet,
ARP, ICMP, IPv4, TCP, and checksum primitives.

## Namespace

- Source components: `Livt.Web.Http`
- Test components: `Livt.Web.Tests.Http`

## Dependencies

`Livt.Net 1.1.0-dev` provides the reusable network-layer primitives used by this package.
The development manifest resolves the sibling checkout. Protocol parsing binds
bounded providers; checksum/header helpers are static. AXI and hardware driver
fixtures belong in Net or the board application, not Web.

## Component Inventory

| File | Component | Role |
|------|-----------|------|
| `src/http/HttpGetRootRecognizer.lvt` | `HttpGetRootRecognizer` | Recognizes the minimal HTTP token `"GET / "` at a given frame offset |
| `src/http/HttpRequestRecognizer.lvt` | `HttpRequestRecognizer` | Recognizes a configured GET token in a bounded TCP payload |
| `src/http/HttpResponseGenerator.lvt` | `HttpResponseGenerator` | Emits bytes for an HTTP/1.0 200 OK response with a fixed-length body |
| `src/http/HttpResponseFrameComposer.lvt` | `HttpResponseFrameComposer` | Composes a complete Ethernet/IPv4/TCP/HTTP response frame byte by byte |
| `src/http/HttpServer.lvt` | `HttpServer` | Coordinates the narrow IPv4/TCP/HTTP request-response subset with TCP handshake |
| `src/http/NetworkEndpoint.lvt` | `NetworkEndpoint` | Delegates Net services and dispatches TCP/HTTP to HttpServer |
| `src/http/WebServer.lvt` | `WebServer` | Application-facing endpoint facade |

## Public API and ownership

- `IHttpContent : IPacketData` selects a stable body with `TrySelect(route)` and
  releases it with `Release()`. Bind one exclusive endpoint owner.
- `NetworkEndpoint<C>` and `WebServer<C>` take content first, then local MAC/IP/port.
- `HttpServer<S, C>` takes borrowed Ethernet/IPv4 parsers, content and local
  MAC/IP/port. The composition root owns and publishes their shared source.
  `NetworkEndpoint` uses one RAM capture for Net and HTTP; HTTP owns only TCP
  parsing. Close TCP with `InvalidateRequest()` before another parser user runs.
- Configure paths with `SetRoutePath`. Load an initialized capture between
  `BeginFrame` and `HandleFrame`. Read only a prepared response using
  `TryReadResponse(index, value)` and check PacketDataResult.
- `BeginFrame` invalidates response graphs before releasing selected content.
  Keep content unchanged throughout preparation and emission.
- `HttpResponsePayload<P>` binds IPacketData and emits HTTP headers plus body.
  `HttpResponseFrameComposer<P>` prepares the shared Net TCP/IPv4/Ethernet graph.
  No raw request arrays, externally supplied checksums or injected body bytes.
- Low-level HttpResponseGenerator remains a header encoder; the frame path uses
  its header bytes and computes TCP checksums over the actual combined stream.
- HttpRequestRecognizer remains a payload-only recognizer; HttpServer owns
  endpoint and TCP flag checks.

The manifest lists the full suite. Tests cover complete independent HTTP frames,
odd/even header and body lengths, bounds, provider read failure and recovery,
plus existing SYN/ACK, route and ARP/ICMP dispatch scenarios.

## In Scope

- HTTP/1.0 GET / recognition over Ethernet/IPv4/TCP
- HTTP/1.0 200 OK response generation with variable Content-Length
- Full Ethernet/IPv4/TCP/HTTP response frame composition
- Minimal TCP handshake (SYN → SYN-ACK, ACK → established, GET / → HTTP response)
- One connection at a time, one listening port
- Fixed response body supplied by the caller

## Out of Scope

- IPv6, UDP, TLS, QUIC
- TCP options, SACK, window scaling, congestion control
- Multiple simultaneous connections
- dynamic application routing beyond configured short GET route slots
- POST, chunked encoding, compression, keep-alive
- DHCP
- Application-owned content stores (those belong in application components)

## Known Constraints

See `COMPILER.md` in this project for Livt compiler workarounds.

## Recommended Next Steps

- Keep application-specific route meaning and content stores in app packages.
- Extend `HttpServer` with a configurable TCP sequence number strategy.
- Add tests for Content-Length values in the 1, 2, 3, and 4-digit ranges.

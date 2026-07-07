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

`Livt.Web` depends on `Livt.Net 0.26.0` for Ethernet, ARP, ICMP, IPv4, TCP,
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
to seven bytes. `HttpRequestRecognizer` combines HTTP matching with fixed
Ethernet/IPv4/TCP checks for a local IP address and TCP port.

### Response Generation

`HttpResponseGenerator` emits an `HTTP/1.0 200 OK` response with
`Content-Type: text/html` and a decimal `Content-Length`. The caller supplies
body bytes one at a time. `HttpResponseFrameComposer` wraps those response bytes
in Ethernet/IPv4/TCP headers by using `Livt.Net` builders and checksum helpers.

### Server And Endpoint Flow

`HttpServer` keeps the small TCP handshake state and selects route-slot
responses. Route metadata is configured by the caller:

```livt
SetRoutePath(route, path)
SetBodyConfig(route, length, checksumWordSum)
AcceptedRoute(route)
```

`NetworkEndpoint` adds ARP and ICMP handling around `HttpServer`. `WebServer` is
a compact facade over `NetworkEndpoint` for applications that want a single
web-facing component.

The common loaded-frame flow is:

1. `BeginFrame()`
2. `LoadRxByte(index, value)` for each received byte
3. `HandleFrame()`
4. `HasResponse()`
5. `GetResponseLength()` and `GetResponseByte(index, httpBodyByte)`

## Scope

In scope:

- HTTP/1.0 GET recognition over fixed-header Ethernet/IPv4/TCP frames
- Route slots for short configured paths
- HTTP/1.0 200 OK response frames with caller-owned body bytes
- Minimal TCP SYN, ACK, PSH+ACK flow for one connection at a time
- ARP and ICMP endpoint dispatch through `Livt.Net`

Out of scope: IPv6, UDP application protocols, TLS, QUIC, TCP options, SACK,
window scaling, congestion control, chunked encoding, compression, keep-alive,
POST handling, dynamic content stores, and multiple simultaneous connections.

## Build And Test

```sh
livt test
```

The configured test list is defined in [`livt.toml`](livt.toml). To force test
regeneration without deleting synchronized dependencies:

```sh
rm -rf out .livt/src.json .livt/ghdl
livt test
```

## Development Notes

- Keep Ethernet, ARP, IPv4, ICMP, TCP, checksum, and frame-I/O primitives in
  `Livt.Net`.
- Keep HTTP recognition, response generation, TCP/HTTP server state, and web
  endpoint dispatch in `Livt.Web.Http`.
- Keep route meaning, page content, and board integration in application
  packages.
- Prefer explicit frame offsets and one-byte-at-a-time builders; that shape maps
  cleanly to hardware and keeps tests small.
- Document compiler workarounds only while they remain reproducible.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).

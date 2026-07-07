# AGENTS.md — Livt.Web

## Project Goal

Provide reusable HTTP/1.0 application-layer components for FPGA projects in the
`Livt.Web.Http` namespace. The package builds on `Livt.Net` for Ethernet,
ARP, ICMP, IPv4, TCP, and checksum primitives.

## Namespace

- Source components: `Livt.Web.Http`
- Test components: `Livt.Web.Tests.Http`

## Dependencies

`Livt.Net 0.26.0` provides the reusable network-layer primitives used by this package.

## Component Inventory

| File | Component | Role |
|------|-----------|------|
| `src/http/HttpGetRootRecognizer.lvt` | `HttpGetRootRecognizer` | Recognizes the minimal HTTP token `"GET / "` at a given frame offset |
| `src/http/HttpRequestRecognizer.lvt` | `HttpRequestRecognizer` | Recognizes a complete IPv4/TCP/HTTP GET / request on a configured port |
| `src/http/HttpResponseGenerator.lvt` | `HttpResponseGenerator` | Emits bytes for an HTTP/1.0 200 OK response with a fixed-length body |
| `src/http/HttpResponseFrameComposer.lvt` | `HttpResponseFrameComposer` | Composes a complete Ethernet/IPv4/TCP/HTTP response frame byte by byte |
| `src/http/HttpServer.lvt` | `HttpServer` | Coordinates the narrow IPv4/TCP/HTTP request-response subset with TCP handshake |

## Public API

### `HttpGetRootRecognizer`

```livt
public fn IsGetRoot(frame: byte[128], offset: int) bool
```

### `HttpRequestRecognizer`

```livt
public const TCP_PAYLOAD_OFFSET: int = 54
public fn IsGetRootRequest(frame: byte[128], localIp: byte[4], localPort: byte[2]) bool
```

### `HttpResponseGenerator`

```livt
public const ETHERNET_IPV4_TCP_HEADER_LENGTH: int = 54
public const FIXED_HEADER_LENGTH_WITHOUT_DIGITS: int = 62
public const CONTENT_LENGTH_DIGIT_OFFSET: int = 58
public const HEADER_CHECKSUM_PREFIX_WORD_SUM: int = 585762
public const HEADER_CHECKSUM_SUFFIX_EVEN_WORD_SUM: int = 6676
public const HEADER_CHECKSUM_SUFFIX_ODD_WORD_SUM: int = 5133
public fn GetHeaderLength(bodyLength: int) int
public fn GetResponseLength(bodyLength: int) int
public fn GetResponseFrameLength(bodyLength: int) int
public fn GetBodyFrameOffset(bodyLength: int) int
public fn GetIpv4TotalLength(bodyLength: int) int
public fn GetTcpLength(bodyLength: int) int
public fn GetHeaderChecksumWordSum(bodyLength: int) int
public fn GetResponseByte(index: int, bodyLength: int, bodyByte: logic[8]) logic[8]
public fn GetResponseByteAtFrameOffset(index: int, bodyLength: int, bodyByte: logic[8]) logic[8]
public fn GetHeaderByte(index: int, bodyLength: int) logic[8]
```

### `HttpResponseFrameComposer`

```livt
public fn GetFrameLength(bodyLength: int) int
public fn GetFrameByte(
    request: byte[128],
    index: int,
    bodyLength: int,
    localMac: byte[6],
    localIp: byte[4],
    ipv4Checksum: byte[2],
    tcpSequence: byte[4],
    acknowledgment: byte[4],
    tcpFlags: byte,
    window: byte[2],
    tcpChecksum: byte[2],
    httpBodyByte: logic[8]) logic[8]
```

### `HttpServer`

```livt
public const RESPONSE_NONE: int = 0
public const RESPONSE_HTTP: int = 2
public const RESPONSE_SYN_ACK: int = 3
public const SYN_ACK_RESPONSE_LENGTH: int = 60
public const CONN_CLOSED: int = 0
public const CONN_SYN_RECEIVED: int = 1
public const CONN_ESTABLISHED: int = 2
public fn SetBodyConfig(route: int, length: int, checksumWordSum: int)
public fn SetRoutePath(route: int, path: int)
public fn AcceptedRoute(route: int) bool
public fn BeginFrame()
public fn LoadRxByte(index: int, value: byte)
public fn HandleFrame()
public fn HasResponse() bool
public fn AcceptedHttpRequest() bool
public fn GetResponseKind() int
public fn GetResponseLength() int
public fn IsHttpBodyFrameIndex(index: int) bool
public fn GetHttpBodyIndex(index: int) int
public fn GetResponseByte(index: int, httpBodyByte: logic[8]) logic[8]
```

## Test Coverage

### `HttpGetRootRecognizerTest`

| Test function | Description |
|---------------|-------------|
| `DetectsGetRootAtTcpPayload` | Frame with "GET / " at offset 54 → `IsGetRoot` returns true |
| `RejectsOtherPath` | Frame with "GET /x" → `IsGetRoot` returns false |
| `RejectsPost` | Frame with "POST /" → `IsGetRoot` returns false |

### `HttpRequestRecognizerTest`

| Test function | Description |
|---------------|-------------|
| `AcceptsMinimalIpv4TcpHttpGetRootRequest` | Full IPv4/TCP/HTTP GET / frame with correct endpoint → returns true |
| `RejectsWrongEndpointOrPayload` | Wrong IP or port or payload → returns false |

### `HttpResponseGeneratorTest`

| Test function | Description |
|---------------|-------------|
| `ExposesResponseLengths` | `GetHeaderLength`, `GetResponseLength`, `GetResponseFrameLength` with bodyLength=64 |
| `EmitsHttpHeaderBytes` | First 15 bytes match "HTTP/1.0 200 OK", Content-Length digits correct |
| `EmitsBodyAfterHeader` | Bytes at and after header offset return supplied bodyByte |
| `EmitsResponseAtFrameOffset` | `GetResponseByteAtFrameOffset` returns 0x00 before offset 54 and correct bytes within |

### `HttpResponseFrameComposerTest`

| Test function | Description |
|---------------|-------------|
| `ExposesFixedResponseFrameLength` | `GetFrameLength(64)` == 54 + 64 + 64 |
| `BuildsEthernetIpv4AndTcpHeaders` | Frame bytes 0–53 contain correct Ethernet/IPv4/TCP header values |
| `AppendsHttpResponseBytes` | Frame bytes from offset 54 contain HTTP response |

### `HttpServerTest`

| Test function | Description |
|---------------|-------------|
| `IgnoresArpRequest` | ARP frame → `HasResponse()` false |
| `RespondsToHttpGetRootRequest` | SYN → SYN-ACK; ACK → no response; GET / → HTTP response with correct length |
| `CalculatesSynAckChecksumForHighTcpBytes` | High-byte TCP seq/ACK → verifies SYN-ACK checksum bytes |

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

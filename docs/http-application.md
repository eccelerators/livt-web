# HTTP application lifecycle and network adapter

`HttpApplication<S, T>` coordinates the bounded parser, router and one owned
response encoder. Its transport boundary is `IHttpApplication : IPacketData`:
request storage is bound at construction, and published response bytes are a
complete HTTP message. The core has no Ethernet offsets, TCP flags, AXI ports,
receive RAM or application body store.

`HttpNetworkEndpoint<S, A>` adapts the existing Net services and TCP packet
composition graph to this boundary. `HttpServer<S, T, CAPACITY>` provides the
default assembly, owning router/application/adapter around an externally shared
source/parser/route graph. WebApp uses this assembly; the legacy APIs are removed.

## Composition and ownership

The application composition root creates one receive provider, one parser graph,
route paths/handlers, a router, and an application. The parser and routes must
refer to the exact same request instance. The graph has one serialized owner and
one clock/reset domain. Constructor arguments are borrowed; there is no runtime
component selection or per-route receive/response buffer.

For HTTP-only tests or another transport:

```text
bounded published source → HttpRequestParser<S>
                              ↓ borrowed by matchers/routes
                          HttpRouter<T>
                              ↓
HttpApplication<S, T>(request, router) → complete HTTP bytes
```

The application owns `HttpResponseEncoder<HttpRouter<T>>`; do not create a second
encoder around it. MAX_BYTES/MAX_TARGET/MAX_HEADERS generic arguments match its
borrowed parser, defaulting to 1024/128/16. The adapter passes a complete-message
response budget to Begin. A selected response that exceeds it cannot publish.

For the Ethernet adapter the composition root additionally constructs:

```text
one RamPacketData<CAPACITY> → EthernetFrameParser<S> → Ipv4PacketParser<...>
                                                        ↓
                                                TcpConnectionRecognizer<S>
                                                        ↓ borrowed HTTP source
                                                HttpRequestParser<...>
```

`HttpNetworkEndpoint<S, A, CAPACITY>(source, ethernet, ipv4, connection,
application, localMac, localIp, localPort)` borrows those same source/parser
instances and the application. It owns Net ARP/ICMP services and one
`HttpTcpTransport<S, A>`; that transport owns a single TCP/IPv4/Ethernet response
graph shared by SYN and HTTP responses. `HttpTransportPayload<A>` selects either
an empty SYN body or already encoded HTTP bytes, without copying a body. No AXI
or particular FrameIo driver appears in this API.

The root owns/publishes/releases receive RAM. CAPACITY sizes the Net ICMP service;
choose it consistently with the actual receive provider and your supported
capture size. The provider must contain the complete declared IP/TCP payload.
The compiling composition in [HttpNetworkEndpointTest](../tests/http/HttpNetworkEndpointTest.lvt)
uses one 256-byte RAM. Larger real browser requests need an appropriately sized
capture and HTTP parser bound; a short captured prefix is rejected.

## Core state and operations

| Operation | Contract |
|---|---|
| Begin(maxResponseBytes, receiveComplete) | Parse and select once, then begin the handler; retries only while Idle/AwaitingRequest |
| Poll | Retry Matching or resume Preparing without repeating accepted handler actions |
| IsReady / GetAvailableLength / TryRead | Complete HTTP bytes while Published or Retained |
| MarkEmitted | Record emission handoff; keep the publication and all borrows for retention/re-read |
| Complete | Only from Retained, after every dependent reader has stopped and invalidated its views |
| Abort | Stop pending/published work after downstream readers stop; cancel handler and close views |
| Release | Return terminal/idle/waiting state to Idle; refuse active matching/preparation/publication/retention |
| GetRequestResult | Preserve the precise parser result until Release |

If receiveComplete is false, Begin returns AwaitingRequest without parsing or
starting a handler. An unpublished source also yields AwaitingRequest. The caller
can finish accumulation and retry Begin because no request transaction was
accepted. Once parsing succeeds, the budget is frozen and repeated Begin cannot
restart work or change it.

The receiveComplete flag means that the transport has delivered its entire
candidate message extent; it does not prove HTTP syntax. The parser must still
find the complete request line and header terminator. Incomplete, malformed,
unsupported, oversized or unreadable HTTP input becomes Failed without running
handlers. This initial adapter aborts that request; it does not synthesize 400/500
responses for parser or handler failures. The caller can inspect GetRequestResult.

Matching may wait for an unpublished configured path. Preparing may wait for the
selected handler. Poll resumes those exact operations. Failed handlers do not
restart and do not fall through to another route. Failed preparation publishes
no bytes. A published read failure clears publication and propagates its original
PacketDataResult with zero output. A bounds error preserves a valid publication.

Complete and Abort close encoder, router/handler, and request view in that order.
Release is also idempotent cleanup, but never releases storage owned by the
caller. Request views remain borrowed through pending work and response retention,
since handlers are allowed to reference request data. Stop all external consumers
before calling completion/cancellation operations. Underlying provider release
and republish cannot substitute for invalidating borrowers.

## Adapter call order

1. Obtain Success from `endpoint.TryRelease()` before releasing/clearing and
   filling the receive store. Publish the complete capture.
2. Call `HandleFrame()`. It handles ARP/ICMP first, then the narrow TCP path.
   Repeated handling of the same active frame returns its existing state.
3. If Pending, call Poll until Prepared or a terminal outcome. Do not load another
   frame or reparse the shared protocol graph while the application is pending.
4. Read a Prepared response through checked `TryRead`. `GetResponseKind()` uses
   typed Arp/IcmpEcho/TcpSynAck/Http values; there are no numeric HTTP route slots.
5. After the transmitter finishes its first emission, call MarkEmitted. The state
   becomes Retained and bytes remain readable. This notification alone does not
   release the response, handler, or receive store.
6. After *all* retained consumers finish, call Complete. The TCP adapter invalidates
   Ethernet/IPv4/TCP descendants before completing the HTTP application. ARP/ICMP
   completion releases their Net service response.
7. Call TryRelease, then release/reuse the receive store. TryRelease returns Busy
   for Pending, Prepared and Retained. Failure/cancellation must stop transmitters
   before Abort and TryRelease.

A transmitter that retains nothing after emission can call MarkEmitted and
Complete consecutively. The API does not infer peer acknowledgement from local
emission and does not promise TCP delivery. Software reset is stopped-consumer
Abort followed by TryRelease; hardware reset must reset the entire common-domain
graph and its external consumers together.

## Narrow TCP subset

This adapter retains the demo's SYN → ACK/combined PSH+ACK → one response behavior,
with fixed initial sequence 1, a fixed 20-byte TCP header, and a FIN-bearing HTTP
response. It binds the peer MAC/IP/port at SYN, rejects other peers, and refuses
HeaderOnly/truncated payload captures. HTTP bytes must fit the fixed-header
Ethernet budget (1460 bytes, supplied by the transport rather than the HTTP core).

It is not a reliable TCP session engine: receive checksums and ACK/sequence-window
validation are not enforced by the existing recognizer; there is no MSS/options
negotiation, segmented-request accumulation, retransmission scheduler or FIN-ACK
closure tracking. These are separate Net transport work, not HTTP handler policy.
The adapter's Retained state provides an ownership boundary for future transport
retention; it does not implement those reliability features by itself.

## Verification

The HTTP-only suite tests phases, waiting/retry, immutable snapshots, cancellation,
budget rejection, parser failures, checked reads, retention and storage reuse.
The adapter suite uses the real shared-RAM parser graph and compares independent
full expected frames, including IPv4/TCP checksums, HTTP bytes, ARP and ICMP. It
also covers capture truncation, incomplete HTTP, peer binding, checksum-preparation
failure, emission failure and sequence-number wrap.

[HttpApplicationTest](../tests/http/HttpApplicationTest.lvt) and
[HttpNetworkEndpointTest](../tests/http/HttpNetworkEndpointTest.lvt) are executable
Livt examples. Verification uses Livt tests only; no synthesis, board build or
flash is implied.

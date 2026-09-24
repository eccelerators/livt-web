# Livt.Web development

Livt.Web is a bounded HTTP application layer over Livt.Net 1.1.0-dev (a sibling
path dependency during development). Source namespace: Livt.Web.Http; tests:
Livt.Web.Tests.Http. Keep application URLs, content and counters in consumers.

## Design

Follow `../livt-book/internal/design/DESIGN_GUIDE.md`. Use tabs, aligned full block
documentation and comments grouping fields by ownership. Explain bounds, failure
results, publication, acceptance vs completion, and borrowed storage lifetimes.
Keep explicit state mutations and preserve scheduled control flow during refactors.
Read `COMPILER.md` before changing compiler workarounds.

## Composition and ownership

`HttpServer<S, T, CAPACITY>` owns HttpRouter/HttpApplication/HttpNetworkEndpoint;
it borrows source, Ethernet/IPv4/TCP parsers, request and routes. The request and
all routes must bind the exact same graph. Source capacity and CAPACITY must agree.
Advanced users compose those layers directly; alternate transports bind
IHttpApplication. The HTTP core has no AXI or packet offsets.

Routes borrow immutable path providers through HttpPath. One selected handler
publishes metadata/body through IHttpHandler; one encoder consumes it. Begin
accepts actions once, Poll resumes, End closes views without undoing accepted work.
No numeric route slots, per-route packet copies or externally injected checksums.

The owner acquires/publishes RX, handles/polls, retains it through response reads
and TX admission/completion, then MarkEmitted/Complete/TryRelease before storage
reuse. On failure stop consumers before Abort/TryRelease. A local emission is
not proof of remote delivery. Keep status snapshots stable until readers stop.

## Tests and limits

List all test components in livt.toml. Preserve independent complete HTTP/frame
expectations, malformed/truncated input, bounds, pending/failure recovery,
retention and cancellation. Use Livt tests only unless synthesis is explicitly
requested. Do not infer area or timing from source shape or simulation.

The bundled TCP adapter is single-peer/single-request-segment/single-response,
with a 1460-byte HTTP budget and no reliable TCP session engine. The parser
default is 1024 HTTP bytes. Document supported behavior in README and docs;
do not link internal project issue tracking from public package documents.

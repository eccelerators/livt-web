# Routes, handlers and fallback responses

The routing components in `Livt.Web.Http` consume one published `IHttpRequest`
and expose one selected `IHttpResponse`. They compose with the bounded parser
and response encoder; they contain no TCP, Ethernet or AXI logic. The current
server retains its earlier routing path until migration to the new
[application/network adapter](http-application.md).

## Composition

An application owns the request source/parser, immutable path providers, body
providers and handlers. Bind each matcher and route to the same request instance.
Bind a handler to one route owner for an active transaction. Routes and chains
are borrowed by one router, and one shared encoder borrows that router.

A single route needs no chain. For example, with a published `ArrayPacketData<128>`
request source, immutable `AppPath : IPacketData` path and `Page : IPacketData` body:

```livt
request: HttpRequestParser<ArrayPacketData<128>>
path: AppPath
page: Page
matcher: HttpPath<HttpRequestParser<ArrayPacketData<128>>, AppPath>
handler: StaticContent<Page>
route: Route<HttpRequestParser<ArrayPacketData<128>>, HttpPath<HttpRequestParser<ArrayPacketData<128>>, AppPath>, StaticContent<Page>>
```

Bind these in the composition root:

```livt
request = new HttpRequestParser<ArrayPacketData<128>>(source)
path = new AppPath()
page = new Page()
matcher = new HttpPath<HttpRequestParser<ArrayPacketData<128>>, AppPath>(request, path)
handler = new StaticContent<Page>(page, HttpContentType.Html)
route = new Route<HttpRequestParser<ArrayPacketData<128>>, HttpPath<HttpRequestParser<ArrayPacketData<128>>, AppPath>, StaticContent<Page>>(HttpMethod.Get, request, matcher, handler)
```

`AppPath` and `Page` denote application components, not package types. Their
bytes can come from constant arrays with checked indexed reads, or from existing
published storage. The framework never copies a route path or page into another
buffer. `HttpPath<R, P, MAX_PATH>` defaults to a 128-byte path bound; choose an
explicit bound appropriate for your configured paths.

Wrap a route in `HttpRouter<RouteType>`, or combine routes as
`RouteChain<FirstRoute, SecondRoute>(first, second)` and wrap that chain in
`HttpRouter<ChainType>`. A chain itself implements `IHttpRoute`, so either child
may be another chain. The types in these short expressions denote complete
concrete generic types; they are not a new type-alias syntax.
[HttpRoutingTest](../tests/http/HttpRoutingTest.lvt) contains a compiling nested
five-route composition with one parser and one `HttpResponseEncoder<RouterType>`.

## Path matching and selection

`HttpPath` validates a configured path on each match: nonempty, starts with `/`,
within MAX_PATH, valid encoded target bytes and complete percent escapes, no
query delimiter or fragment. The provider and parsed request must both be ready.
Configuration/read errors return Failed rather than a route miss. An unpublished
input returns NotReady. A match borrows the providers for the duration of the
operation; configuration must remain immutable while routing is active.

Matching uses the parsed path exactly. `/status?x=1` selects the `/status` route;
`/status/`, `/Status` and `/%73tatus` do not. There is no percent decoding, case
folding, dot-segment removal or repeated-slash collapsing. Applications can
implement `IHttpRouteMatcher` for other matching policies without changing the
router. Custom matchers must preserve the same checked-read and error contract.

`Select()` does no handler work. Routes are visited in tree order:

1. Select the first route whose path **and method** match; stop scanning there.
2. A matching path with the wrong method does not hide a later valid method.
3. If no route is selected, at least one path match yields MethodNotAllowed.
   Union the supported methods from all matching paths and remove duplicates.
4. If no path matches, return NotFound, including for an unsupported request method.
5. Failed/NotReady matching stops scanning. Do not turn either into 404/405.

Route methods are Get or Head. Other is a parsed request classification, not a
wildcard route method; configuring it is an error. HEAD routing/Allow metadata
is supported for composition, but the current response encoder does not suppress
HEAD bodies. The initial production application should expose GET routes only;
a transport/application extension is required before serving HEAD responses.

A route freezes its selection result until End. Only NotReady is retryable, over
the same unchanged request. A router may retry Select while NotReady without
starting handlers. After selection, repeated Select returns the same result.
End clears every child's cached outcome, including path-only matches and misses,
so the next request cannot inherit an earlier 404/405 decision.

## Handler lifecycle

`IHttpHandler : IHttpResponse` adds Begin, Poll and End. A handler returns Idle
before Begin, Pending while preparing, then Prepared or Failed until End.
Repeated Begin must not perform another application action. Poll in Idle does
nothing; Poll after a terminal result retains that result. Pending work is
resumed through Poll, not by repeating Begin. Responses must stay immutable from
publication through transport retention. A read failure aborts the response;
it does not select the next route or start an error handler mid-emission.

`StaticContent<P>(body, contentType)` borrows an already published body and serves
200 OK. Begin validates its response metadata and published extent. An unavailable
body fails; it is not silently treated as pending. End invalidates the response
view without releasing application-owned storage.

[CounterContent](../tests/http/CounterContent.lvt) is an executable dynamic-handler
example. It snapshots a digit from a borrowed application service in Begin,
returns Pending, finishes after two Poll calls, and exposes the JSON body `[n]`
through checked reads. Changing the service during pending work or emission does
not change the snapshot. The example's counters/failure control are test
instrumentation. Production handlers can substitute hardware operations or a
bounded application-owned content store while retaining the same lifecycle.
There is no framework JSON serializer or application-state store.

`HttpRouter<T>` coordinates only route selection and the selected handler:

| Operation | Behavior |
|---|---|
| Select | Freeze a selection or fallback outcome, without starting work |
| Begin | Begin selected handler once, or prepare shared fallback |
| Poll | Resume Pending; validate publication of a prepared response |
| IsReady / metadata / TryRead | Expose only a prepared body and typed metadata |
| End | Cancel/complete selected work and clear route outcomes; idempotent |

A failed selected handler remains failed; it cannot fall through to an overlapping
route. Begin before Select or while selection is NotReady returns Idle. Failed
selection produces Failed preparation, not a successful fallback. The caller must
check every result before preparing the encoder.

## Fallbacks and ownership

The router owns one `HttpRouteFallback` with constant plain-text bodies:

- 404: `Not Found\n`, with no Allow metadata.
- 405: `Method Not Allowed\n`, with the union of matching routes' allowed methods.

The shared encoder writes the actual Content-Length, status, media type and Allow
header. Fallback generation needs no body RAM or per-route encoder. A matching
application route can serve custom error content explicitly; generalized fallback
handler injection is not part of this initial API.

Call order is parse, Select, Begin, Poll until Prepared/Failed, prepare encoder,
consume response, stop all transport readers, invalidate encoder, End router,
invalidate request, then release/reuse backing storage. Pending cancellation
stops work with End. End is not rollback for an already accepted application
side effect. Retain borrowed request and body storage until their readers finish.
The graph has one serialized owner and a common reset domain.

The router clears outputs on failed reads and invalidates publication. Bounds
errors leave a valid response published. Provider contract violations such as
reusing storage without invalidating readers cannot be detected reliably without
generation tokens and are prohibited. The [application/transport coordinator](http-application.md) establishes this
ordering for the new network adapter. Demo migration remains a separate step.

## Verification

Livt tests exercise the real parser → matcher → nested routes → router → encoder
composition against independent full HTTP response literals. They also cover path
configuration and bounds, exact encoded matching, query separation, overlapping
methods, first-match ordering, Allow union, fallback reuse, Pending, cancellation,
handler failure, stable snapshots, unavailable inputs and failed reads.
No synthesis or board/resource/timing claim is part of this verification.

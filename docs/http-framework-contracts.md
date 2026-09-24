# HTTP application contracts

The isolated `verification/route-contracts` project is the historical prototype
in `Livt.Web.Prototype`, retained as design evidence. Its interfaces are not the
public package API. The production framework and migrated WebApp use
`Livt.Web.Http`; see [bounded request parsing](http-request-parser.md),
[response encoding](http-response-encoding.md), [routing](http-routing.md), and
[application/transport contracts](http-application.md).

## Composition and ownership

The application is the composition root. It owns request storage, a parser,
immutable path providers, content providers, handlers, routes, the router and
one response encoder. Dependencies passed to constructors are borrowed; only
one owner may call scheduled methods in a shared graph at a time. All components
in the graph share a reset domain. No route owns another HTTP parser or packet
buffer. The router selects a fixed child; no runtime component references or
heterogeneous component arrays are needed.

Actual prototype constructor shapes:

```livt
matcher = new HttpPath<Request, PathBytes>(request, pathBytes)
handler = new StaticContent<Body>(body, HttpContentType.Html)
route = new Route<Request, Matcher, Handler>(HttpMethod.Get, request, matcher, handler)
routes = new RouteChain<FirstRoute, SecondRoute>(firstRoute, secondRoute)
```

`Request`, `PathBytes`, `Body`, etc. denote concrete component types here; the
compiling test contains their complete generic types. Bind the same request to
the matcher and route. This is an ownership invariant, not an identity check
performed at runtime. Stateless matchers may be shared if access is serialized;
stateful handlers have one exclusive route owner during a transaction.

The shorthand `Route(GET, HttpPath("/"), StaticContent(page))` expresses the
intended convenience API. It is **not** claimed as supported Livt syntax. The
prototype validates explicit generic construction. Literal-path convenience
wrappers remain convenience work; the production router now supports nested
RouteChain compositions. See [production routing](http-routing.md).

## Request contract

`IHttpRequest` is a read-only view of a fully parsed, supported request:

| Method | Contract |
|---|---|
| IsReady | True only while the complete request view is published |
| GetMethod | Typed method classification; usable only while ready |
| GetPathLength | Encoded path length, excluding query; usable only while ready |
| TryReadPath | Checked byte access; failure returns no usable value |

The prototype injects parsed metadata using a test fixture. It does not validate
HTTP syntax, recognize a query delimiter or accumulate fragmented requests.
The production parser must establish these facts before publishing the view:

- A complete, supported request line and header terminator were received.
- All views refer to initialized bytes within the declared request bounds.
- The target is origin-form with a nonempty path starting with `/`.
- Split at the first literal `?`; query presence and length are distinct, so a
  missing query differs from an empty query. A literal fragment marker is invalid.
- Validate percent escapes; preserve their encoded bytes. No percent decoding,
  case folding, dot-segment removal, slash collapsing or trailing-slash rewriting.
  `/a`, `/%61` and `/a/` are distinct matching inputs. `%2F` is not a separator.
- Invalid controls, malformed framing, unsupported bodies or limits violations
  cannot be represented as a ready successful request.

The production parser extends the view with version, query presence/length/read
and bounded indexed-header access. The isolated prototype retains its minimal
request fixture until routing is promoted. Header names compare case-insensitively;
values retain their bytes with explicitly documented whitespace rules. Duplicate
and conflicting framing headers require explicit rejection/handling rules;
there is no arbitrary mutable header dictionary. Use a separate parse result
with Ready, Incomplete, Malformed, Unsupported, TooLarge and ReadFailure outcomes.
A GET token alone is never Ready. TCP accumulation is outside this contract.
The production parser classifies Get/Head/Other without implementing method
behavior: a syntactically valid bodyless non-GET request can be Ready so routing
can decide 405 versus a missing path. Bodies/framing outside the parser subset
remain Unsupported. The initial application handlers remain GET-only.

## Matching and ordering

`IHttpRouteMatcher.TryMatch()` returns Match, NoMatch, NotReady or Failed.
`HttpPath<R, P>` compares equal-length encoded path bytes from two borrowed
providers, sequentially. A read failure is not a route miss. Route path providers
must be stable, nonempty, valid encoded paths without query delimiters. Their
construction/validation belongs to application configuration, outside matching.

The isolated prototype `IHttpRoute` separates TryMatchPath from AllowsMethod and
handler methods. The production `IHttpRoute` instead exposes Select and
GetMatchedMethods plus the handler contract, allowing the same interface for
leaves and arbitrarily nested RouteChain subtrees.
The router selects the first route for which **both** path and method match.
An earlier same-path route for another method does not hide a later valid route.
If all routes are examined with no selection, at least one path match means
MethodNotAllowed; no path match means NotFound. NotReady and Failed stop the scan
rather than triggering a fallback handler.

The production router aggregates supported methods for the 405 Allow header;
the isolated prototype only demonstrates the distinct dispatch outcome. It does not encode
404/405 responses. Duplicate path/method definitions have deterministic first
priority; they must not run multiple handlers. HEAD appears in the prototype
as a dispatch discriminator, not evidence of implemented HTTP HEAD semantics.
The initial production application remains GET-only.

## Handler and response contract

`IHttpHandler : IHttpResponse : IPacketData` exposes these operations:

| Operation | Meaning |
|---|---|
| Begin | Start once for a selected request; return Pending, Prepared or Failed |
| Poll | Resume the selected operation without restarting application actions |
| End | End completed/failed work or cancel pending work; idempotent cleanup |
| IsReady | True only when the stable response is Prepared |
| GetStatus / GetContentType | Typed metadata, usable only while ready |
| GetAvailableLength / TryRead | Published body bytes only, without HTTP headers |

Idle is the initial/released state. Poll in Idle returns Idle. Repeated Begin
returns the existing state without repeating actions. Prepared and Failed are
terminal until End; failure must not cause route fallback or automatic restart.
Begin does not accept a new request while active. Select while selected returns
the existing selection; it does not reconsider matching until End.

The owner retains request views while selection/preparation is active. Handlers
may borrow them until End; an optimization releasing them earlier requires an
explicit stronger contract. Body bytes and response metadata remain stable from
Prepared through End. Dynamic handlers snapshot mutable inputs before publishing.
StaticContent borrows an already published immutable body; End clears its own
publication and does not release the application's underlying storage.

The prototype status set is Ok, NotFound, MethodNotAllowed and
InternalServerError, with Html, PlainText and Json media kinds. These are typed
semantic values, not enum ordinals to emit on the wire. The production response encoder
maps them to status codes and fixed text, derives Content-Length from the published
body, and validates the adapter-supplied byte budget. Parser-related errors may
extend the set. The encoder is shared across routes.

The production response contract now adds `GetAllowedMethods()` with None, Get,
Head and GetAndHead metadata, and BadRequest/Unknown status plus Unknown content
type. A 405 requires nonempty Allow metadata; other supported statuses require
None. The isolated prototype retains its earlier minimal interface as historical
contract evidence; production routing implements the extended response interface. The implemented encoder emits HTTP/1.0, explicit close and
Content-Length over its actual borrowed body; it does not implement HEAD body
suppression. See the response encoding document for exact APIs and limits.

Read failures remain PacketDataResult values. Consumers must ignore output bytes
unless the result is Success. Failed preparation exposes no body. An emission
read failure aborts transfer; do not replace missing bytes with zeros or generate
a second HTTP response after transmission has begun.

## Completion, cancellation and reset

Normal flow: publish request, select, Begin, Poll until Prepared, prepare the
encoder/transport, emit, wait for transport retention to end, invalidate response
views, End, invalidate request views, then release/reuse backing storage.

Abort follows the same dependent-before-provider ordering. Stop scheduled reads
and cancel the transport before invalidation and End. End cancels internal work
but does not promise rollback of already accepted external effects. Handlers must
not repeat such effects in Poll. The pending-handler test counts Begin actions.

The transport decides when retained bytes are no longer needed, which may be
later than the last byte handed to hardware. No retransmission behavior or
reliable TCP implementation is implied by this prototype. Reset must clear
publication/selection and pending control in all graph owners together; surviving
memory contents are not published. Full reset/adapter tests belong to application
coordination, not this isolated scheduled-call prototype.

## Prototype verification scope

Use Livt tests in `verification/route-contracts`. They exercise heterogeneous
handler dispatch, prepared static bytes, checked bounds, stable body borrowing,
pending work, cancellation/reuse, failure isolation and distinct routing outcomes.
They establish language/API feasibility and scheduled behavior only. They do not
establish HTTP parsing, complete wire responses, nested full-router behavior,
LUT cost, RAM mapping, timing closure or board operation.

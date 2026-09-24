# Route contract prototype

This isolated Livt project proves the proposed HTTP component bindings and
scheduled lifecycle before replacing the existing server. Read
`../../docs/http-framework-contracts.md` for ownership, matching and failure
contracts. Nothing here is exported by Livt.Web's main manifest.

From the Livt.Web checkout:

```sh
cd verification/route-contracts
livt test -f
```

The project uses the sibling Livt.Net checkout. Only Livt tests are needed.
There is no synthesis or hardware programming step.

The test fixture owns one request byte store and injects parsed metadata.
`HttpPath` borrows that request and an immutable `RootPath` byte provider.
Two generic routes bind different handler types: `StaticContent` and a test
handler that returns Pending before publishing `{}`. A second chain exercises
same-method priority; chains sharing children are driven exclusively.

Tests cover body bytes/bounds, provider lifetime, asynchronous preparation,
cancellation and reuse, method mismatch versus missing path, read failure,
failed preparation, duplicate-route ordering and supplied query boundaries.
The query test does not parse HTTP: the fixture injects the path extent.

This is deliberately a two-leaf composition prototype. Production nested
routing, fallback response encoding, Allow aggregation, request parsing, reset
coordination and transport integration remain separate implementation steps.
No hardware resource or full HTTP conformance claim follows from these tests.

## Verification result (2026-09-24)

`env -u _JAVA_OPTIONS livt test` passed **8 tests, 0 failed, 0 skipped**
with default optimizations in an isolated copy at `/tmp/web001/prototype`.
Simulation completed at 67,305 ns. The promoted Livt sources differ only in
blank-line cleanup; the manifest uses the equivalent relative sibling Net path.
The local dependency checkout was Livt.Net commit `63fe5a84a54c62b77425e5da5feab93a3dd57556`.
Evidence for this run is `/tmp/web001/prototype-test.log`.

The original fixture assumed per-method reset; explicit cleanup was added when
that assumption failed. The passing fixture now ends both exclusive chains,
invalidates the borrowed request, releases published buffers and resets its
observation counter before each test. No compiler defect was found.

The existing Web server sources and main manifest are unchanged. Its integration
suite was not rerun for this isolated prototype. No synthesis or board runs were
performed.

## Production successor

The production components now live in `Livt.Web.Http`; see
[route composition and handlers](../../docs/http-routing.md). This isolated
prototype retains its original minimal request/response interfaces and two-leaf
chain as historical compilation evidence. New consumers should use the production
API, which supports nested chains, the full parsed request, and typed Allow metadata.

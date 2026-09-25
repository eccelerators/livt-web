# FPGA resource and timing verification

The 2026-09-25 Arty A7-100T integration exposed three source-level hotspots.
Measurements below belong to the complete WebApp/Net/Web board design, not a
standalone cost for Livt.Web.

## Source changes

- `HttpResponseEncoder` formats Content-Length using ten decimal places and at
  most nine scheduled subtractions per place. It preserves zero, the entire
  nonnegative 32-bit range, overflow rejection and exact header bytes without a
  wide signed division/remainder path.
- `HttpTcpSequence` uses one unsigned 32-bit addition followed by network-order
  byte extraction. It avoids the former unrolled signed carry/division chain.
- `HttpRequestHeaders` stores offsets/lengths using `Bits.RequiredFor(MAX_BYTES)`
  bits and exposes them through the same zero-extended integer getters. Stable
  descriptors are captured before scan loops. Default raw table payload is 704
  bits instead of 2,048, excluding generated shadows and dispatch logic.

No routes, request limits, validation rules or response bytes were removed.

## Functional verification

169 HTTP-core checks pass using the deployment compiler snapshot. This includes
all prior core tests plus four TCP arithmetic boundaries and one exact 300-byte
request with a 273-byte header value and later header offsets above 255.
The 46 encoder cases retain their independent complete-header fixtures, including
ten-digit lengths, exact budgets, provider failures and overflow rejection.

The unchanged, slow HttpNetworkEndpointTest suite was excluded. The previously
deferred full-frame simulation is not claimed as passing. To reproduce the core
run, use an isolated copy with all production sources, the normal test fixtures,
and every manifest test component except HttpNetworkEndpointTest; omit that test
source from the copy. Run `livt test --events -v` there.

CLI SHA-256:
`59f9d15d5d4855e311a44890978df0ebbdd0af8a5c12a94571d0c3a3384cfafb`.
VHDL extension SHA-256:
`468b73b6422137d75a61d91b6ea6968292e5aa9eaf30da0586255496234c5cfe`.
GHDL uses its GCC backend. No generated production HDL was edited.

## Hardware results

Default implementation exceeded LUT capacity. ExploreArea reduced LUTs enough
but failed slice packing. Following it with ExploreSequentialArea allowed
placement/routing: 40,114 LUTs, 61,026 registers, 15,610 slices and 16 BRAM tiles.
That baseline still failed 100 MHz setup timing at -9.500 ns; hold slack was
+0.050 ns. The build correctly refused to produce a bitstream.

The worst path was the encoder's decimal division. Further failing paths included
TCP sequence advancement (-7.944 ns) and header descriptor updates (-5.643 ns).
With the same area settings, the final complete board design uses 36,643 LUTs
(57.80%), 56,247 registers (44.36%), 15,209 slices (95.96%) and 16 BRAM tiles.
This saves 3,471 LUTs, 4,779 registers and 401 slices against the routed baseline.
At 100 MHz, routed setup slack is +1.252 ns and hold slack +0.011 ns; all
user-specified timing constraints are met without the obsolete encoder exception.
Slice packing remains close to capacity despite the lower LUT count.

The bitstream was flashed, verified and booted successfully. UART startup and
route logging work. Eight complete HTTP checks pass: Home, About, status, query
routing, missing path, trailing slash, unsupported method and Home again.
Exact headers and bodies are checked, with only validated status-counter digits
masked. Socket checks do not cover the deferred raw-frame simulation assertions.
No route, request limit or response content was reduced to obtain this result.

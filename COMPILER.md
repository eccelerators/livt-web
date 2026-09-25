# Active compiler workarounds

## Header classifier table selection (#542)

HttpRequestHeaders.GetExpectedNameByte keeps the enum-driven constant-array
selection outside the classifier loop. The equivalent inline if/elif assignment
chain followed by the branching lowercase comparison generates duplicate FSM
choices in the affected compiler. A lowercase local alone does not avoid it.
The extracted helper preserves comparison behavior; no generated VHDL is edited.

## Scheduled result field forwarding (#543)

HttpRequestCursor.Begin captures GetAvailableLength into a local, validates that
local and stores it as the cursor length. Directly assigning the scheduled return
to a field after resetting that field leaves a stale forwarding variable in the
affected compiler; a subsequent capacity check can incorrectly accept oversize
input. The local result is required until the compiler regression is fixed.

Both regressions are recorded internally with standalone Livt tests. The complete
parser/capacity suite passes with these source-level workarounds and default
optimizations. Re-test the original forms after compiler fixes before removing
these notes. No package-wide optimization changes are required.

## Context-free TCP sequence helper (#544, #545, #547)

HttpTcpSequence now uses one unsigned 32-bit addition and unsigned shifts to
extract network-order bytes. This replaces the former ascending carry loop,
whose signed divide/remainder chain failed the board's 100 MHz timing target.
The six independent sequence tests cover byte carry, full wrap, zero advance,
the signed boundary, a full payload and the largest nonnegative count.
The historical descending-loop (#544) and unsigned-divisor (#545) compiler
regressions remain tracked; the current helper needs neither construct.

Initialize the local byte array with explicit indexed assignments. An array
initializer containing parameter expressions in a context-free function is
silently omitted from generated VHDL (#547), corrupting the TCP acknowledgement.
The explicit-assignment helper passes independent carry/wrap tests with warnings
as errors. Keep the independent complete-frame expectations strict.

Reduction also found parameterless context-free functions emitting illegal empty
input records (#546). Current production helpers take real inputs and do not
need a dummy parameter workaround. Standalone regressions for all four issues
are recorded internally in the compiler verification repository.

## Scheduled elif fall-through (#550)

HttpNetworkEndpoint.Complete returns immediately after ARP/ICMP completion, then
handles TCP in a separate branch. An elif condition containing a scheduled call
can run after the earlier branch was taken, incorrectly attempting TCP completion
for a network-service response. The reduced two-branch fixture passes with this
explicit-return form. Keep both ARP/ICMP and TCP completion checks.

## Constructor-local component imports (#553)

HttpServer and the consumer composition roots declare owned parser/router/route
nodes as explicit typed fields. Constructor-local component allocations forwarded
to borrowed references can omit their package imports in generated root VHDL,
causing missing input-record types at elaboration. The local minimal reproduction
fails; the equivalent field form passes. Keep ownership explicit until fixed.

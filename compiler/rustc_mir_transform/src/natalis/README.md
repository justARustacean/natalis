# Natalis M1: read-only MIR SLP analysis

## Status

This is a design RFC for the private `natalisStream` branch. It intentionally
does **not** register a MIR pass, add a `-Z` option, change a query, or alter
code generation. It is the contract for the first implementation slice.

Natalis investigates superword-level parallelism (SLP) in Rust while preserving
Rust's ownership, aliasing, panic, drop, and unwinding semantics. M1 is an
analysis and reporting facility only. It does not transform MIR, vectorize
operations, create alias guarantees, or expose a user-facing language feature.

## Placement in the MIR pipeline

The eventual M1 analysis runs after borrow checking and
`run_analysis_cleanup_passes`, while the body is in
`MirPhase::Analysis(AnalysisPhase::PostCleanup)`. It runs before
`run_runtime_lowering_passes` and before `run_optimization_passes`.

This boundary is deliberate:

- borrow checking has already accepted or rejected the program;
- post-borrowck cleanup has removed analysis-only false edges and normalized
  the CFG enough for deterministic inspection;
- runtime lowering, inlining, scalar replacement, propagation, and CFG
  simplification have not erased the source-adjacent statement structure that
  makes a rejected or accepted candidate explainable.

The implementation is an optional, disabled-by-default, read-only MIR pass.
It receives `&Body<'tcx>` semantically: it must neither mutate the body nor
derive facts that require mutation. Its future opt-in report is internal and
will be selected by `-Znatalis-report` through MIR-dump infrastructure.

## Internal model

The names in this section are the planned internal vocabulary, not public
compiler APIs.

### `NatalisCandidate`

A candidate represents an ordered pack of at least two assignments in one
`BasicBlock`.

```text
NatalisCandidate {
    body: DefId,
    block: BasicBlock,
    statements: [Location],       // increasing statement index, same block
    operation: OperationShape,    // e.g. Add<i32>, Mul<f32>
    scalar_ty: Ty,
    reads: [PlaceSummary],
    writes: [PlaceSummary],
    control_flow: ControlFlowContext,
    status: Eligible | Rejected(Reason),
}
```

`OperationShape` includes the MIR binary operator, scalar type, overflow
mode, and any semantic flag necessary to distinguish operations. A pack never
mixes operators or scalar types. `PlaceSummary` describes the root local and
the allowed projection path; it is not an address and is not a provenance or
aliasing claim.

An `Eligible` result means only “within this layer's proof rules, this pack
may be passed to a later target-aware cost model.” It never implies that a
backend will vectorize it. `Rejected(Reason)` records the first canonical
reason the pack cannot enter the layer, rather than silently dropping it.

### `NatalisCandidateReport`

A report is a deterministic, per-MIR-body collection of candidates and
rejections. Its future pretty-printer will be a MIR-dump sidecar, not a
diagnostic, lint, stable flag, or source-visible annotation.

Report traversal order is fixed:

1. body order is the compiler query traversal order;
2. blocks use ascending `BasicBlock` index;
3. candidates use the first statement `Location` in ascending order;
4. ties use the ordered lane `Location` sequence;
5. reads, writes, and rejection reasons use their declared canonical order.

The report must include enough MIR identity to correlate an entry with a normal
MIR dump, but must not include pointer addresses, hash iteration order, or
target-dependent cost decisions. Future MIR-opt tests compare this textual
artifact.

### `ControlFlowContext`

Every candidate includes, but is not formed across, CFG context:

```text
ControlFlowContext {
    predecessors: sorted [BasicBlock],
    successors: sorted [BasicBlock],
    dominance: Entry | Dominated | LoopHeader | Join | Other,
    has_unwind_edge: bool,
}
```

The classification is descriptive in M1. A candidate cannot include statements
from a successor, predecessor, sibling, or loop iteration. `Join` describes a
block with multiple reachable incoming edges; `LoopHeader` describes a
natural-loop header when the CFG can establish one. Ambiguous or irreducible
CFG structure is `Other`, never a reason to infer lane masks.

## Candidate layers

M1 tries layers in order. A later layer never relaxes a preceding layer's
safety requirements.

### Layer 1: scalar locals

Recognize independent `Rvalue::BinaryOp` or `Rvalue::CheckedBinaryOp`
assignments when all operands are `Operand::Copy` of scalar local places and
all destinations are scalar local places. Initial scalar types are integer and
floating-point primitive types of the same concrete `Ty`.

The assignments must have the same `OperationShape`, preserve the original
statement order, and have pairwise-disjoint read/write local roots. Checked
arithmetic is rejected until a later layer proves the same overflow behavior
and panic/check ordering lane by lane.

Accepted shape:

```text
bb0:
    _2 = Add(copy _0, copy _1);
    _5 = Add(copy _3, copy _4);
```

### Layer 2: non-packed fields

Extend a Layer 1 operation to places whose projection consists only of field
projections rooted at distinct compiler-known local allocations. Tuples and
non-`repr(packed)` structs are eligible. The final scalar field type and
binary operation must still match exactly.

Accepted shape:

```text
bb3:
    (_0.0: i32) = Add(copy (_1.0: i32), copy (_2.0: i32));
    (_3.1: i32) = Add(copy (_4.1: i32), copy (_5.1: i32));
```

The analysis does not infer disjointness merely because field names differ.
It needs distinct local roots, or a future, separately specified projection
disjointness proof.

### Layer 3: constant indices

Extend the Layer 2 rules to index projections only when every index is a
compile-time constant, the root local allocation is compiler-known, bounds are
proven, and all read/write locations are non-overlapping. Dynamic indexing,
slice length calls, and any index whose bounds check could panic remain
ineligible.

Accepted shape:

```text
bb7:
    _0[0] = Add(copy _1[0], copy _2[0]);
    _3[1] = Add(copy _4[1], copy _5[1]);
```

The first implementation may conservatively report a rejection until the
constant-index and bounds proof is implemented; it must never promote the
example to eligible merely because the syntax resembles this shape.

## Mandatory rejections

Natalis fails closed. Any unknown fact is `MayAlias` and prevents eligibility.
M1 rejects a prospective pack containing any of the following:

| Condition | Canonical reason |
| --- | --- |
| call, virtual call, tail call, or call terminator dependency | `Call` |
| drop, storage lifetime effect, or unwind-sensitive operation | `DropOrUnwind` |
| atomic or volatile access | `AtomicOrVolatile` |
| inline assembly | `InlineAsm` |
| raw-pointer dereference or unknown dereference | `UnknownProvenance` |
| dynamic index or unproven bounds | `DynamicOrUncheckedIndex` |
| packed field | `PackedField` |
| move operand or moved/dead source | `MoveOrInvalidPlace` |
| different operation shape or scalar type | `IncompatibleOperation` |
| read/write overlap or uncertain aliasing | `MayAlias` |
| checked/panic-capable behavior without an order proof | `ObservableCheckOrder` |
| statements in different basic blocks | `CrossBlock` |

Examples of rejection:

```text
_2 = Add(move _0, copy _1);                  // MoveOrInvalidPlace
_3 = Add(copy (*_4), copy _5);               // UnknownProvenance
_6 = Add(copy _7[_8], copy _9[0]);           // DynamicOrUncheckedIndex
_10 = Add(copy _11, copy _12); Call(...);    // Call
```

## Alias and allocation-region roadmap

M1 consumes only facts it can justify from the MIR shape and existing ownership
model. Its default relation is `MayAlias`; absence of evidence never becomes
`Disjoint`.

A later flow-sensitive analysis may introduce allocation-region identities for
compiler-known local allocations. A region ends at `StorageDead`, a move, or
an escaping borrow. Heap allocations, FFI, raw pointers, pinning, and unknown
deref bases begin outside the model and stay `MayAlias`.

Future parameter-alias summaries are facts at MIR locations and CFG edges, not
function-wide assumptions. At a CFG join, a fact survives only when every
incoming reachable edge supports it; otherwise it widens to `MayAlias`.
Calls and escaping references invalidate precision until their individual
semantics are proved. These facts must complement, never replace, borrow
checking.

## Control-flow progression

M1 observes CFG context but creates no polymorphic pack and no lane mask.
Subsequent stages are intentionally ordered:

1. retain M1 single-block candidate reports;
2. add explicit, conservative alias/region dataflow;
3. recognize canonical loops using dominance and back-edge evidence;
4. model divergent branches with abstract masks and defined reconvergence;
5. lower only target-supported, cost-effective candidates to backend vector
   IR.

No stage may cross a control-flow edge, hoist a panic, reorder a drop, or
change unwind behavior without a proof specific to that operation and target.

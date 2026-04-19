# Overview

Entry to the design folder. This page orients the reader to the layer stack, treats the cross-cutting concepts that span more than one per-layer page, and flags what's settled versus what's open. For the short worldview and numbered invariants see [`../../ARCHITECTURE.md`](../../ARCHITECTURE.md); the per-layer pages are the depth treatment for each layer.

## Layer composition

The layers form a stack. Each layer is generic over the one beneath it:

```
Application                 (user logic, not a trait in the stack)
    ↑
Partitioning                sharding, replication, network IO as state machine
    ↑
CRDTStore                   convergence; generic over a foldable event trait
    ↑
RelationalStore             secondary indices, schema-as-data
    ↑
KVStore                     ordered key-value lookup structures
    ↑
PageStore                   raw byte storage; the side-effect boundary
```

The generic relationship is straightforward: `KVStore<P: PageStore>`, `RelationalStore<K: KVStore>`, and so on. Implementations are chosen at composition time and resolved at compile time. Monomorphization specializes the full stack into a single concrete type, and the compiled binary has no vtable lookups or indirect calls across layer boundaries.

What the architecture commits to is that the trait bounds compose: each layer's trait is parameterized on the one below it, and the resulting generic type checks out end to end. It does not commit to any specific pattern for layer method signatures; each layer's interface is whatever it needs to be to serve its role. The per-layer pages state what each layer's methods are.

The layer count is not fixed. Additional layers may exist between Partitioning and the application boundary — caching, permissions, schema migration, materialized view maintenance — and each composes over the layer beneath by the same generic-over-trait pattern.

## Properties of the stack

Three properties hold throughout:

1. **Synchronous.** A step runs to completion before the next step is accepted. There is no `async`, no threading, no channel, no blocking call anywhere in the stack.
2. **Deterministic.** Given the same sequence of inputs, the stack produces the same sequence of state transitions and outputs. There is no hidden randomness, no time dependence, no concurrency inside.
3. **Total.** Every input to the stack produces a transition. A step that has nothing to do returns `None` where the signature has an `Option` return; the state machine still advances. Truly fatal conditions (see the error model below) are separate.

There is no universal wrapper around every return type. `Option<T>` is a per-method choice.

Asynchrony is not excluded from the system; it is excluded from the stack itself. Time, IO, concurrency, and network transport live outside, in an external runtime that feeds inputs in and dispatches outputs out. See [`partitioning.md`](partitioning.md) for how this composes at the top of the stack and [`sync.md`](sync.md) for sync-over-network as state-machine IO.

## Side effects and portability

Externally observable mutation is confined to `PageStore` — see ARCHITECTURE.md invariant 1 for the rule and [`pagestore.md`](pagestore.md) for the layer-level treatment. The composition-level consequence: porting OrigamiDB to a new platform — bare metal, an embedded flash sector, a browser's linear memory, a remote object store — requires implementing a single trait. Every layer above the `PageStore` boundary is unchanged across those targets.

## Error model at layer composition

ARCHITECTURE.md invariant 4 states the rule: no soft errors; if it is not fatal, recover. What composition adds is that every condition that looks like an error has a distinct architectural home.

- **Normal operation with an optional return.** A method whose signature declares `Option<T>` returns `None` when the operation has nothing to produce (e.g., a lookup for a missing key). The state machine advances; nothing failed.
- **Rejected at the security boundary.** Malformed, unauthenticated, or protocol-invalid input is dropped at the network boundary, before entering the Partitioning state machine. Only verified inputs pass through. See [`partitioning.md`](partitioning.md).
- **Truly fatal.** The process terminates. State persists synchronously through the stack, so restart reconstructs the upper layers from what `PageStore` holds; see [`fold.md`](fold.md).

There is no `Result<T, E>` on the hot path. Soft-error variants propagate silently and branch every caller; the architecture prefers loud failures that restart against silent corruption.

## Cross-cutting concepts

Three concepts span multiple layers. Each is treated in depth on its own page; this is the orientation.

### Linearize-during-sync

Distributed systems that need replicas to agree on state traditionally take one of two paths. Consensus-based systems (Raft, Paxos) coordinate across replicas before a write is considered committed — each operation must be agreed on by a quorum before it is durable and visible. Traditional CRDT systems take the opposite approach: each peer accepts writes immediately without coordination, and the system relies on properties of the operations (commutativity, associativity, idempotence) and per-operation metadata (vector clocks, tombstones, causal context) to guarantee convergence.

OrigamiDB takes a third path: peers accept writes without coordination, and concurrent operations from peers are merged into a deterministic total order at the sync boundary.

Downstream of sync, the fold runs over an already-totally-ordered log. This collapses the fold's contract to a sequential step: apply each event in order. Commutativity is not required of individual operations because operations arrive pre-ordered. The complexity that traditional CRDTs carry in every operation's metadata is paid once, at the sync boundary, and all peers agree on the result because the ordering is deterministic.

See [`sync.md`](sync.md) for the protocol mechanics and [`../adr/0001-linearize-during-sync.md`](../adr/0001-linearize-during-sync.md) for the rationale.

### The fold constructs state

The fold is the bridge between the write-optimized representation (the linearized operation log) and the read-optimized representation (the materialized state). It runs on the write path, processing each event in turn and building indexed, structured, queryable state. Reads do not touch the log; reads query the state the fold constructed.

See [`fold.md`](fold.md) for the full treatment and [`../adr/0002-fold-constructs-state.md`](../adr/0002-fold-constructs-state.md) for the rationale.

### Security at the IO boundary

Authentication, integrity verification, and encryption live at the boundary between the Partitioning layer and the external network. Every layer below that boundary operates on trusted, authenticated data. The invariant: verify at the boundary, trust everything inside. See [`partitioning.md`](partitioning.md) for the layer-level treatment.

## Open questions (spec-wide)

- The trait naming above CRDTStore is unsettled. This spec uses **Partitioning** for the layer that owns sharding, replication, and network IO. The name is working, not final.
- The exact boundary between the Partitioning layer and the security layer — whether security is a layer or a sub-concern inside Partitioning — is undecided. This spec treats security as a concern sitting at the IO boundary without committing to whether it has its own trait.
- Whether additional layers (caching, permissions, migration) warrant their own pages in this spec, or remain application-layer concerns that compose over Partitioning, is open.
- The specifics of the foldable event trait, the linearization algorithm in sync, and several per-layer details are pending specification. Each is flagged in the relevant per-layer page's own open-questions section.

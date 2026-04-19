# Overview

This page describes how the layers of OrigamiDB compose, the properties the stack as a whole satisfies, and the cross-cutting concepts that appear at more than one layer. It is the foundation the per-layer pages build on.

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

The generic relationship is straightforward: `KVStore<P: PageStore>`, `RelationalStore<K: KVStore>`, and so on. Implementations are chosen at composition time and resolved at compile time. The trait boundaries exist at design time, not at runtime — monomorphization specializes the full stack into a single concrete type, and the compiled binary has no vtable lookups or indirect calls across layer boundaries.

What the architecture commits to is that the trait bounds compose: each layer's trait is parameterized on the one below it, and the resulting generic type checks out end to end. It does not commit to any specific pattern for layer method signatures; each layer's interface is whatever it needs to be to serve its role. The per-layer pages state what each layer's methods are.

The layer count is not fixed. Additional layers may exist between Partitioning and the application boundary — caching, permissions, schema migration, materialized view maintenance — and each composes over the layer beneath by the same generic-over-trait pattern.

## Properties of the stack

Three properties hold throughout the stack:

1. **Synchronous.** A step runs to completion before the next step is accepted. There is no `async`, no threading, no channel, no blocking call anywhere in the stack.
2. **Deterministic.** Given the same sequence of inputs, the stack produces the same sequence of state transitions and outputs. There is no hidden randomness, no time dependence, no concurrency inside.
3. **Total.** Every input to the stack produces a transition. Conditions that look like errors are either normal (an operation rejected by a predicate, returning `None` where a method's signature has an `Option` return) or truly fatal (hardware failure — see the error model below).

There is no universal wrapper around every return type.

Asynchrony is not excluded from the system; it is excluded from the stack itself. Time, IO, concurrency, and network transport live outside, in an external runtime that feeds inputs in and dispatches outputs out. Intended side effects of higher layers bubble up by return and may allow stateful continuations — structurally equivalent to what `Future::poll` and regenerator-runtime compile into under the hood.

## Side effects are confined to PageStore

All externally observable mutation in the stack happens at exactly one layer. Every trait method across the stack takes `&mut self`, and upper layers may mutate their own internal state during an operation — what they may not do is perform mutation that anyone outside the stack can observe. Only `PageStore` does that.

This is the property that enables stub-free testing. Swap in an in-memory `PageStore` implementation and every layer above runs unchanged. The upper layers are deterministic functions over their arguments and the `PageStore` state beneath them; nothing outside the stack observes their internal mutations. Effect reasoning falls out of the trait surface without needing effect types in the language.

Porting OrigamiDB to a new platform — bare metal, an embedded flash sector, a browser's linear memory, a remote object store — requires implementing a single trait. Every layer above the `PageStore` boundary is unchanged.

## Networking as returned intent

Networking enters the stack at the upper layers, but the layers themselves do not perform IO. The Partitioning layer receives inbound packets as inputs to its state machine, processes them like any other operation, and may produce outbound packets as outputs. An external runtime transports those bytes on whatever medium the deployment uses — TCP, UDP, Bluetooth, WebSocket, a mock in tests.

This is the returned-intent pattern: the stack computes what IO should happen and returns that as data. Something outside the stack performs the IO and feeds the results back in as inputs. The pattern keeps the stack pure and testable without requiring algebraic effects (which Rust does not have) or async coloring (which the stack's synchronous contract excludes).

The same pattern applies to sync messages between peers, to security-layer verification, and to any other outbound action the stack needs to take. See [`sync.md`](sync.md) and [`partitioning.md`](partitioning.md) for how this composes at the upper layers.

## Error model

The stack does not have soft errors. If a condition is not fatal, it is recovered from, not returned up the call stack as an error variant.

Three categories exhaustively cover every condition that looks like an error:

- **Normal, rejected by a predicate.** An operation is valid but is not applied — for example, an invariant predicate in the fold skips an operation that would violate a cross-peer constraint. The state machine advances; the output is `None` where the signature has an `Option` return.
- **Rejected at the boundary.** Malformed, unauthenticated, or protocol-invalid input is rejected at the network boundary, before entering the Partitioning state machine. The security layer sits between Partitioning and the external network; only verified inputs pass through.
- **Truly fatal.** Hardware failure, disk failure, flash burnout, the operating system lying about persistence. The process terminates. Recovery is a restart; how the stack reconstructs its state on restart is a cross-cutting recovery-strategy question, with pure log replay as the baseline (see [`fold.md`](fold.md)).

Restarting from a known-good checkpoint is a form of recovery, not a fatal condition. Everything between normal operation and hardware-level catastrophe is recoverable by restart.

There is no `Result<T, E>` on the hot path. Soft-error variants propagate silently and branch every caller; the architecture prefers loud failures that restart against silent corruption.

## Cross-cutting concepts

Three concepts span multiple layers. Each is treated in depth on its own page; this is the orientation.

### Linearize-during-sync

Distributed systems that need replicas to agree on state traditionally take one of two paths. Consensus-based systems (Raft, Paxos) coordinate across replicas before a write is considered committed — each operation must be agreed on by a quorum before it is durable and visible. Traditional CRDT systems take the opposite approach: each peer accepts writes immediately without coordination, and the system relies on properties of the operations (commutativity, associativity, idempotence) and per-operation metadata (vector clocks, tombstones, causal context) to guarantee that any two peers that have received the same set of operations arrive at the same state, regardless of the order those operations arrive.

OrigamiDB takes a third path: peers accept writes without coordination, and concurrent operations from peers are merged into a deterministic total order at the sync boundary.

Downstream of sync, the fold runs over an already-totally-ordered log. This collapses the fold's contract to a sequential step: apply each event in order. Commutativity is not required of individual operations because operations arrive pre-ordered. The complexity that traditional CRDTs carry in every operation's metadata is paid once, at the sync boundary, and all peers agree on the result because the ordering is deterministic.

The specifics of how the causal DAG is constructed and linearized are pending detailed specification in [`sync.md`](sync.md).

### The fold constructs state

The fold is the bridge between the write-optimized representation (the linearized operation log) and the read-optimized representation (the materialized state). It processes the log and builds indexed, structured, queryable state. Reads do not touch the log; reads query the state the fold constructed.

Precise trait shape and mechanics are pending specification. What is settled: the fold is a write-path transformation that produces read-optimized state, not a read-time query mechanism. See [`fold.md`](fold.md) for the full treatment.

### Security at the IO boundary

Authentication, integrity verification, and encryption live at the boundary between the Partitioning layer and the external network. Inbound bytes are verified and decrypted before the Partitioning state machine ever sees them. Outbound bytes are signed and encrypted before transmission.

This concentrates security in one layer with a well-defined surface. Every layer below the security boundary operates on trusted, authenticated data. The architectural invariant is the same as the network-level trust boundary in a TEE or a firewall: define the boundary once, verify at the boundary, trust everything inside.

See [`partitioning.md`](partitioning.md).

## Open questions (spec-wide)

- The trait naming above CRDTStore is unsettled. This spec uses **Partitioning** for the layer that owns sharding, replication, and network IO. The name is working, not final.
- The exact boundary between the Partitioning layer and the security layer — whether security is a layer or a sub-concern inside Partitioning — is undecided. This spec treats security as a concern sitting at the IO boundary without committing to whether it has its own trait.
- Whether additional layers (caching, permissions, migration) warrant their own pages in this spec, or remain application-layer concerns that compose over Partitioning, is open.
- The specifics of the foldable event trait, the linearization algorithm in sync, and several per-layer details are pending specification. Each is flagged in the relevant per-layer page's own open-questions section.

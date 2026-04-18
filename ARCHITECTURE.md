# Architecture

OrigamiDB is an embeddable database with a distributed consensus model drawing from ideas used in CRDTs. It is built in layers of composed Rust traits. Porting to a new storage medium requires implementing one trait at the bottom of the stack.

This document is the worldview. It states the architectural invariants and names the layers and their boundaries. For deeper specification see `docs/spec/`. For the proposal process see `rfcs/`. For guidance specific to LLM-assisted contributors see [AGENTS.md](AGENTS.md).

## Architectural invariants

These hold across the entire stack. Implementations that violate any of them are wrong, regardless of how well they perform.

1. **Mutations are confined to the bottom layer.** Only `PageStore` holds mutable references to page data. Every layer above operates on shared references. Rust's borrow checker enforces this at compile time.

2. **Every trait method is a synchronous state machine step.** The shape is `(&mut self, input) -> Option<Output>`, equivalently `(state, input) -> (state, maybe_output)`. No `async`, no threads, no channels, no blocking inside any layer.

3. **Asynchrony lives outside the stack.** An external runtime feeds inputs in and dispatches outputs. The stack itself never waits. At the upper layers, networking enters as inbound packets that trigger operations; outbound packets are returned as outputs that an external loop transmits.

4. **Soft errors are not permitted.** Operations either complete (producing a new state and optionally an output) or the process terminates and restarts from a known-good checkpoint. There is no `Result<T, E>` on the hot path. Conditions that look like errors are either normal (an operation skipped by an invariant predicate in the fold) or unrecoverable (disk failure, memory exhaustion, logic bug — terminate and restart).

5. **Convergence is achieved by linearizing during sync.** Concurrent operations from peers are merged into a deterministic total order at the sync boundary, using a two-layer tiebreaker: deterministic mechanical ordering (peer id, logical timestamp) plus the semantic meaning of the operation. Once linearized, the fold downstream is a trivially sequential scan.

6. **The fold constructs state.** The fold processes the linearized operation log and produces read-optimized state. It is not a query mechanism. Reads do not touch the log; reads query the constructed state.

## The layer stack

Each layer is a Rust trait, generic over the layer beneath it. Implementations are swappable independently.

- **`PageStore`** — raw byte-level reads and writes against the underlying medium. All mutations are confined here. A `no_std` implementation can operate over a pre-allocated memory region; a Linux implementation can be a memory-mapped file; any blob store is a candidate.

- **`KVStore`** — ordered key-value lookup structures over `PageStore`. The choice of data structure (LSM tree, B-tree, sorted array, skip list) is an implementation detail of this layer. The value associated with a key is opaque to the KVStore.

- **`RelationalStore`** — secondary indices, schema, and structural metadata over the key-value layer.

- **`CRDTStore`** — merge logic and convergence. Generic over a merge algorithm. Linearizes concurrent operations from peers into a deterministic total order during sync.

- **`Partitioning`** — sharding, replication, and cluster membership. Membership metadata is replicated through the convergence layer beneath it; shard assignments are computed deterministically from that metadata. Networking enters the stack at this layer, but the layer does not perform IO — it computes what IO should happen.

The layer count is not strictly fixed. Additional layers may exist between `Partitioning` and the application boundary as the design matures.

## Crate structure

The core traits and reference implementations are intended to live in a single `no_std`-compatible crate. Per-platform `PageStore` implementations are pushed out as separate crates.

The repository is pre-implementation. This is the planned shape, set so that the trait surface stays portable to constrained environments without dragging platform-specific allocator or IO dependencies.

## Cross-cutting concerns

**The interface shape is universal.** Every layer shares the `(&mut self, input) -> Option<Output>` signature. The same shape describes in-process trait calls and stateful network protocols at the packet layer.

**Mutations-confined-to-PageStore is enforced by the type system.** Upper layers hold `&` references to page data; only `PageStore` holds `&mut`. The architectural invariant is mechanical, not disciplinary.

**Networking is returned intent.** The Partitioning layer produces outbound packets as state machine outputs. An external runtime reconciles intent with side effect — opening sockets, transmitting bytes, delivering inbound packets back as state machine inputs. The core never performs IO.

**Multiple folds over the same log.** Different fold functions over the same linearized operation log produce different read-optimized projections. Adding a new projection adds no overhead to the write path.

**Frontier cache as garbage collection boundary.** A snapshot of constructed state at the most recent point all peers have confirmed (the "frontier") allows reads to fold forward only over operations since the frontier. Operations behind the frontier are folded into the cache and become discardable. Garbage collection is a side effect of sync, not a separate process.

**Security at the IO boundary.** Authentication, integrity verification, and encryption sit between `Partitioning` and the external network. Inbound bytes are verified before reaching the state machine; outbound bytes are signed and encrypted on the way out.

## Where to find more

- `docs/spec/` — the specification (in progress)
- `rfcs/` — proposal process for substantive changes
- [`AGENTS.md`](AGENTS.md) — guidance for LLM-assisted contributors
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to participate

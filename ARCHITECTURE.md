# Architecture

OrigamiDB is an embeddable database with a distributed consensus model drawing from ideas used in CRDTs. It is built in layers of composed Rust traits. Porting to a new storage medium requires implementing one trait at the bottom of the stack.

This document is the worldview. It states the architectural invariants and names the layers and their boundaries. For deeper specification see `docs/spec/`. For guidance specific to LLM-assisted contributors see [AGENTS.md](AGENTS.md).

## Architectural invariants

These hold across the entire stack. Implementations that violate any of them are wrong, regardless of how well they perform.

1. **Side effects are confined to the bottom layer.** Only `PageStore` performs externally observable mutation. Layers above take `&mut self` on their own trait methods and own an implementation of the layer beneath them; they may mutate their own internal state during an operation, but remain deterministic over their arguments and the `PageStore` state beneath. This is what enables stub-free testing — swap in an in-memory `PageStore` and every upper layer runs the same way — and gives disciplined effect reasoning in a language without effect types.

2. **Every trait method is a synchronous state machine step.** The shape is `(&mut self, input) -> Output`, equivalently `(state, input) -> (state, output)`. `Output` may be `Option<T>` on a per-method basis when the operation has an optional return; it is not universal. No `async`, no threads, no channels, no blocking inside any layer.

3. **Asynchrony lives outside the stack.** An external runtime feeds inputs in and dispatches outputs. The stack itself never waits. At the upper layers, networking enters as inbound packets that trigger operations; outbound packets are returned as outputs that an external loop transmits.

4. **Soft errors are not permitted.** If it is not fatal, recover. Restarting from a known-good checkpoint is a form of recovery, not a fatal condition. There is no `Result<T, E>` on the hot path; conditions that look like errors are either normal (an operation rejected by an invariant predicate in the fold, returning `None` where the signature has one) or truly fatal — hardware failure, disk crashes, flash burnout, the operating system lying about persistence. Everything between those is recoverable by replay, resync, or checkpoint restore.

5. **Convergence is achieved by linearizing during sync.** Concurrent operations from peers are merged into a deterministic total order at the sync boundary. Once linearized, the fold downstream is a trivially sequential scan. The specifics of how the causal DAG is constructed and linearized are pending detailed specification.

6. **The fold constructs state.** The fold processes the linearized operation log and produces read-optimized state. It is not a query mechanism. Reads do not touch the log; reads query the constructed state. Precise trait shape and mechanics are pending specification.

## The layer stack

Each layer is a Rust trait, generic over the layer beneath it. Implementations are swappable independently.

- **`PageStore`** — raw byte-level reads and writes against the underlying medium. All externally observable mutation is confined here. A `no_std` implementation can operate over a pre-allocated memory region; a Linux implementation can be a memory-mapped file; any blob store is a candidate.

- **`KVStore`** — ordered key-value lookup structures over `PageStore`. The choice of data structure (LSM tree, B-tree, sorted array, skip list) is an implementation detail of this layer; the intended first implementation is B-tree. The value associated with a key is opaque to the KVStore.

- **`RelationalStore`** — secondary indices, schema, and structural metadata over the key-value layer. Whether schema can be enforced at compile time (eliminating runtime checks on the hot path) is an open design question.

- **`CRDTStore`** — merge logic and convergence. Generic over a foldable event trait — `fold` lives on the event type itself. Linearizes concurrent operations from peers into a deterministic total order during sync. Trait specifics pending.

- **`Partitioning`** — sharding, replication, and cluster membership. Membership metadata is replicated through the convergence layer beneath it; shard assignments are computed deterministically from that metadata. Networking enters the stack at this layer, but the layer does not perform IO — it computes what IO should happen.

The layer count is not strictly fixed. Additional layers may exist between `Partitioning` and the application boundary as the design matures.

## Crate structure

The core traits live in a single `no_std`-compatible crate along with a handful of `PageStore` implementations behind feature flags: `Vec<u8>` and `[u8; N]` variants useful for testing, file and block-device backed variants behind default flags for users, and most of the alloc-dependent options gated behind `alloc` or `std`. Exotic per-platform implementations (WASM linear memory, S3 and equivalent blob stores) live in separate crates.

The repository is pre-implementation. This is the planned shape, set so that the trait surface stays portable to constrained environments without dragging platform-specific allocator or IO dependencies.

## Cross-cutting concerns

**Side effects at `PageStore` enable effect reasoning.** With externally observable mutation confined to one layer, the rest of the stack is deterministic over its arguments and the `PageStore` state beneath. This makes stub-free testing tractable — swap in an in-memory `PageStore` and every upper layer runs the same way — and gives effect reasoning in a language without effect types.

**Networking is returned intent.** The `Partitioning` layer produces outbound packets as state machine outputs. An external runtime reconciles intent with side effect — opening sockets, transmitting bytes, delivering inbound packets back as state machine inputs. The core never performs IO.

**Security at the IO boundary.** Authentication, integrity verification, and encryption sit between `Partitioning` and the external network. Inbound bytes are verified before reaching the state machine; outbound bytes are signed and encrypted on the way out.

## Where to find more

- `docs/spec/` — the specification (in progress)
- [`AGENTS.md`](AGENTS.md) — guidance for LLM-assisted contributors
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to participate

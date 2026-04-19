# KVStore

KVStore provides ordered key-value lookup structures over `PageStore`. It is the layer where data structures that make point lookup, range scan, and prefix iteration cheap actually live. Everything above the KVStore layer operates in terms of key-value access without knowing or caring how that access is physically implemented underneath.

## Responsibilities

A KVStore implementation:

- Maps keys to values. Keys are byte strings with a defined ordering. Values are opaque byte strings.
- Supports at minimum point lookup, ordered range scan, and prefix iteration.
- Is parameterized by a `PageStore`. All durable state flows through the PageStore boundary.
- Chooses an internal data structure appropriate to its target.

The data structure choice is an implementation detail of the KVStore trait, not a property of the contract. Different deployments can compose different KVStore implementations over the same PageStore without any other layer changing.

## Value opacity

KVStore does not interpret values. A value associated with a key is an opaque byte string; the KVStore layer reads and writes those bytes and imposes no structure on them. The layer above — relational, or fold-constructed — decides what values contain.

This matters more than it sounds. Value opacity is what lets the same KVStore serve very different workloads with very different physical value layouts. A value can be a serialized row, a columnar chunk, an opaque binary record, or anything else. The KVStore provides ordered lookup; it does not prescribe what is ordered.

## Ordering contract

Keys are totally ordered. Range scans and prefix iteration return keys in ascending order. Two calls that iterate the same range with no intervening write return the same sequence.

The ordering is the one the KVStore defines, typically lexicographic byte order. Composite keys encode their component fields so that the encoded byte order matches the intended semantic order (for example, big-endian integer encoding so that numeric order equals byte order).

## What KVStore does not do

- **No interpretation of values.** Values are bytes. Structure lives above.
- **No secondary indices.** Secondary indices, when present, are maintained by the Relational layer, not by the KVStore. From KV's perspective, a secondary index is just another set of key-value pairs.
- **No schema.** There is no concept of a table, column, or type at this layer. Keys and values are bytes.
- **No merge or convergence logic.** Concurrent operations from peers are not resolved at this layer. The CRDTStore layer handles convergence; by the time operations reach KV for application, they have already been ordered.
- **No transactions across multiple operations.** The synchronous state machine contract means one operation is applied at a time. A multi-page atomic write within a single operation is a PageStore concern; cross-operation atomicity is not something KV provides because the state machine makes it unnecessary.

## Implementation strategies

The trait admits several conventional choices. Each has a different profile on the tradeoff between write throughput, read latency, space amplification, and allocation behavior.

- **B-tree.** Balanced. Predictable locality, straightforward read paths, simpler correctness story than LSM. **This is the intended first implementation** per the crate-structure note in [`ARCHITECTURE.md`](../../ARCHITECTURE.md). A B-tree KVStore over a file-backed PageStore is the reference deployment.
- **LSM tree.** Write-optimized. Memtables absorb writes; sorted runs are flushed and later compacted. Appropriate when writes are frequent and the caller tolerates deferred compaction work. (The sort-merge pattern LSM uses internally resembles the sort-merge pattern the sync protocol uses to produce the linearized log. The resemblance is structural, not operational — the two processes live at different layers and serve different purposes.)
- **Skip list.** Simpler than a B-tree, probabilistic balance, easier to keep allocation-free in fixed-size deployments.
- **Sorted array.** Minimal. Ideal for embedded deployments with small working sets, where the entire store fits in one or a few pages and writes are rare.

The spec does not prescribe a choice beyond the first-implementation note. Additional strategies are additive.

## Composition with PageStore

KVStore implementations call into PageStore for all durable reads and writes. The data structure the KVStore implements is laid out across pages; moving pages, splitting nodes, or compacting runs are operations the KVStore performs in its own internal state and commits through PageStore.

Because PageStore is the single externally-observable-mutation boundary, a KVStore implementation does not need to manage its own write-ahead log or crash recovery mechanism independent of PageStore's. Whatever durability PageStore provides propagates upward.

## Allocation-free viability

Several KVStore strategies admit allocation-free implementations suitable for `no_std` targets. A sorted array over a fixed page is the simplest. A small skip list with a pre-allocated node pool is another. A B-tree variant that works within a bounded node pool is possible. An LSM variant that works within a fixed memtable allocation and writes sorted runs of bounded size is more involved but possible.

Implementations that require dynamic allocation expose that requirement through a feature flag or are distributed as separate crates. The core trait does not bake in an allocator dependency.

## Contract and invariants

- **Pure composition over PageStore.** The KVStore layer has no durable state of its own outside what PageStore manages. Any in-memory state (cache, memtable, iterator cursor) is rebuildable from PageStore-backed data on restart.
- **Deterministic ordering.** Given the same keys, any KVStore implementation produces the same iteration order.
- **Opaque values.** A KVStore may not depend on the structure of values it stores. A value byte string is written and read verbatim.
- **Synchronous contract.** A call to the KVStore completes or the process terminates. There is no background thread that performs writes asynchronously from the caller's point of view. (An implementation may defer work internally — LSM compaction is the canonical example — but the deferred work is not visible in the state machine's observable order of operations.)

## Open questions

- **Range scan cursor lifetime.** How iteration handles concurrent writes within the same state machine step (likely not an issue given synchronous execution) versus across steps (likely requires the caller to re-establish the cursor).
- **Snapshot semantics.** Whether KVStore exposes explicit snapshots, or whether read-your-writes within a step is all that is required. The fold-constructs-state model reduces the need for explicit snapshots, but it does not eliminate them for certain read-fold patterns.
- **Value size bounds.** Whether the trait assumes small values (that fit in a page) or accepts arbitrary-size values with the implementation handling spilling. Different targets will want different defaults.
- **Compaction exposure.** For an LSM-backed KVStore, whether compaction is entirely internal or exposed to the caller as work units that can be scheduled externally (relevant for embedded targets with real-time constraints).
- **Key encoding conventions.** Whether the spec prescribes how composite keys are encoded (length-prefixed, separator-based) or leaves it to each caller. Uniform conventions enable cross-layer tooling; rigid conventions limit flexibility.

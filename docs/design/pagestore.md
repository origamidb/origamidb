# PageStore

The bottom layer of the stack. PageStore manages raw byte storage on whatever medium the deployment provides. It is the single place in the architecture where externally observable mutation happens; every other layer operates on trait methods that may mutate internal state, but only PageStore performs mutation that anyone outside the stack can observe.

## Responsibilities

A PageStore implementation provides read and write access to a byte-addressable store of pages. A "page" is the unit of granularity at which the layer hands out reads and accepts writes; its size and structure are an implementation detail of the chosen backend. What the layer guarantees is that the bytes it stores can be read back as written, and that a successful write either persists fully or not at all.

Everything above PageStore reads page data and writes it through PageStore calls. Higher layers may cache pages in their own internal state, but the durable state of the system lives inside PageStore.

## What PageStore does not do

- **No key-value structure.** PageStore does not know about keys, values, or lookup ordering. That is the KVStore layer's concern. A PageStore implementation is free to treat pages as opaque byte blobs addressed by a page identifier.
- **No schema.** Relational concepts, secondary indices, and structured queries live in layers above. Pages are bytes.
- **No operation semantics.** PageStore does not interpret the contents of the pages it stores. Whether a page holds a B-tree node, an LSM run, a columnar chunk, or any other format is invisible to the layer.
- **No concurrency control.** The synchronous state machine contract means PageStore sees one input at a time. It does not need locks, MVCC, or conflict resolution; those are either unnecessary (by construction) or handled above.
- **No network awareness.** Even when backed by an object store, PageStore is invoked by the stack as a pure read-or-write request. Any network IO that implements the backend is the platform runtime's responsibility, returned as intent where relevant.

## Implementation targets

The trait is designed so that substantially different environments can provide a PageStore with the same contract. The anticipated targets include:

- **`Vec<u8>` / `[u8; N]`** for tests and bare-metal `no_std`. A pre-allocated byte buffer is the backing store. No heap allocation required for the `[u8; N]` variant.
- **File-backed** for general-purpose deployments. File on disk, memory-mapped or accessed via direct IO.
- **Block device** for controlled deployments where the backend is a raw block device rather than a filesystem.
- **Flash sector** on embedded devices. Reads return bytes; writes are subject to erase-block constraints that the implementation manages internally.
- **Linear memory** in WebAssembly. A WASM module's heap is the medium; persistence can be composed on top via IndexedDB or equivalent.
- **Object store** (S3 or equivalent) for cold or distributed deployments. Pages are objects; reads are range GETs; writes are PUTs or multipart uploads. IO is returned as intent for the runtime to perform.

Per [`ARCHITECTURE.md`](../../ARCHITECTURE.md), the core traits crate includes implementations for the platform-independent targets — `Vec<u8>`, `[u8; N]`, file, block device — behind feature flags. Exotic platform-specific implementations (WASM linear memory, object stores) live in separate crates.

## Durability and atomicity

PageStore is responsible for durability: if it returns success on a write, the bytes survive a crash. How it achieves this is an implementation choice — write-ahead logging, copy-on-write page allocation, fsync-equivalent primitives, or platform-provided guarantees (battery-backed NVRAM, transactional object stores).

Atomicity of a multi-page write — where either all pages persist or none do — is a contract between PageStore and the layer immediately above. A KV operation that needs to update several pages atomically (a B-tree node split, an LSM flush) submits the change as a batch; PageStore either commits the batch or returns the store to its pre-batch state on crash.

Atomicity across layers is not PageStore's concern. Because the stack is a synchronous state machine, only one operation is ever in flight at a time. A compound change that spans multiple layers is a single step of the state machine; if the step completes, every layer's page-level effects have been committed through PageStore. If the step does not complete, the process recovers from the last durable checkpoint.

## Allocation-free operation

For `no_std` and embedded deployments, a PageStore implementation can be written to avoid heap allocation entirely. The `[u8; N]` backing case is the minimal example — a fixed memory region provided at initialization; reads return slices into that region; writes overwrite bytes in place; no allocator is needed.

This is not a compromise feature. An allocation-free PageStore is not a stripped-down version of the richer one — it is the same trait with a simpler backend. Higher layers compose over it without modification. The absence of an allocator at the bottom of the stack is inherited by the layers above as long as they are also written to be allocation-free.

## Contract and invariants

- **Externally observable mutation is confined here.** Upper layers may mutate their own internal state via `&mut self` on their trait methods, but the durable state of the system — what anyone outside the process can observe by restarting and reading — lives only in PageStore. This property is an architectural contract that implementations uphold; it is not enforced by the type system.
- **Reads return what was written.** A successful write followed by a read of the same page, with no intervening write to that page, returns the written bytes.
- **Either full persistence or no persistence.** A write that returns success has persisted in full. A write that does not return (because the process terminates) leaves the store either fully updated or fully at its prior state; no torn writes are visible after recovery.
- **No visible in-layer caching.** If the implementation caches pages, the cache is internal and does not expose stale reads. Upper layers do not need to know whether a read was satisfied from cache or medium.

## Crash recovery

On process restart, the stack reloads its durable state through PageStore and reconstructs the upper layers from what PageStore holds. PageStore itself is responsible for making the durable state readable after restart. How the upper layers use that durable state to resume operation is a cross-cutting recovery-strategy question tracked in [`fold.md`](fold.md) and in this page's open questions below.

What is fixed: PageStore's contract is the boundary. Everything durable is routed through it, and everything that survives a restart is reconstructable from its contents.

## Open questions

- **Page size.** Fixed across the store, per-store at initialization, or variable per write? Object-store backends favor variable; flash backends favor fixed-and-aligned; file and mmap backends are flexible.
- **Page addressing.** An opaque `PageId` handed out by PageStore, a byte offset, or something higher layers compute? Trade-off between stable identity across page relocations and transparent layout.
- **Batch contract.** How multi-page atomic writes are expressed in the trait. Options include an explicit batch type, a transactional wrapper, or a convention that a single method call is atomic.
- **Iteration.** Whether PageStore exposes ordered iteration over pages it owns, or whether iteration is a pure KVStore concern expressed over random-access reads.
- **Page allocation.** Whether new pages are requested from PageStore (and returned on deletion) or whether the upper layer manages a free list over a fixed region.
- **Crash recovery protocol.** What the upper layers do on restart to reconstruct state — pure log replay (the baseline), checkpoint-based resume, or some other mechanism — is a cross-cutting recovery-strategy concern. The specifics are entangled with log compaction and undo/redo support; see [`fold.md`](fold.md).
- **Error surface.** PageStore runs inside the state machine, where soft errors do not exist. A write that cannot be committed (hardware failure, medium failure) is fatal and causes the process to terminate; the stack then recovers by restart. Whether the trait signature expresses this by returning `()` on success and panicking on fatal failure, or by a richer type that higher layers treat as uninhabited, is a trait-design question.

# RelationalStore

RelationalStore provides structured access over `KVStore`: secondary indices, schema awareness, and the shape of the materialized state that reads query. It is the layer where application-facing structure lives without yet knowing about convergence, sharding, or networking.

## Responsibilities

A RelationalStore implementation:

- Maintains secondary indices over the key-value layer beneath it, so that queries by non-primary fields are served without scanning primary data.
- Holds and applies schema — the description of what structure values have and how they are organized into logical units (tables, documents, or whatever vocabulary the application builds).
- Expresses structural operations (insert, update, delete, index lookup) in terms of the KV operations immediately underneath.
- Preserves the invariants of the layers below while adding its own structural invariants.

The trait sits over KVStore and under CRDTStore. It works with data that has already been ordered by the sync protocol and linearized; it does not participate in convergence.

## Schema is data in the log

Schema in OrigamiDB is not external configuration. A schema change is an operation in the linearized operation log, processed by the fold like any other operation. When the fold encounters a schema-changing operation, it updates its interpretation rules for subsequent operations in the same log.

This has useful properties:

- All peers see the same schema change at the same log position. No version negotiation, no coordinated deployment, no out-of-band migration.
- Concurrent schema changes resolve the same way any other concurrent operations do: they are linearized by the sync protocol and applied sequentially by the fold.
- The fold modifies its own interpretation rules based on the data it processes. The system's behavior is defined by the data it consumes. This is a metacircular pattern, analogous to code that reads a configuration file it just wrote, but grounded in a linear log instead of a file system.

The practical consequence for RelationalStore is that schema state is not a separate persistent artifact. The relational layer reconstructs its current schema by folding operations from the log, the same way it reconstructs any other part of the materialized state. A new peer joining the system receives the schema by receiving operations; no parallel schema channel exists.

## Secondary indices

Secondary indices are the RelationalStore's signature contribution. A secondary index is a mapping from a non-primary field value to the set of primary keys that have that field value. The index is maintained as the fold processes operations — an insert of a row with a given value in the indexed field adds an entry; a delete or update removes or rewrites the entry.

From the KVStore's point of view, a secondary index is just another set of key-value pairs. The relational layer chooses a key encoding that makes the index usable (typically the indexed value followed by the primary key, so range scans on the indexed value return matching primary keys in order).

Secondary indices are not replicated independently. They are a property of constructed state, built by the fold. Two peers with the same linearized log build the same indices; nothing about index maintenance requires cross-peer coordination.

## What RelationalStore does not do

- **No query language.** Relational operations are expressed programmatically through the layer's API. A SQL parser or other surface language is an application-layer concern that composes over RelationalStore, not part of it.
- **No convergence.** Convergence lives in CRDTStore, above. RelationalStore processes operations in the order they arrive from above, without participating in linearization.
- **No sharding.** Partitioning is handled by the layer above CRDTStore. Relational operations at a peer see only the data that peer holds; cross-shard operations are coordinated at the Partitioning layer.
- **No transactional machinery.** Atomicity of a compound operation is a property of the state machine's synchronous execution, not of an explicit begin/commit protocol at this layer. A compound operation is a single input; if it completes, every index update and every primary write committed together.
- **No durable state of its own.** Everything RelationalStore needs is expressed through KVStore. In-memory caches of index metadata or schema state are rebuildable from KV-backed data on restart.

## Composition with KVStore and CRDTStore

The relational layer sits in the middle of the stack, and the texture of its contract is different on each side.

Downward to KVStore, the contract is operational: the relational layer issues KV reads and writes, and expects them to behave as the KV contract specifies. The primary data for a table becomes one set of KV entries; each secondary index becomes another. Encoding conventions for how the relational layer lays keys out over the KV namespace are an implementation choice, not baked into the trait.

Upward to CRDTStore, the contract is structural: CRDTStore submits operations to RelationalStore that have already been linearized by the sync protocol. RelationalStore applies them in order, updates the indices and schema state as appropriate, and does not second-guess the ordering. The layer does not need to know which peer an operation came from or how conflicts were resolved — the ordering already encodes that.

## Relationship to the fold

The RelationalStore's constructed state — indices, schema state, primary data organized by structural layout — is built by the fold. Reads do not scan the operation log; reads query the state the fold constructed. A point lookup by primary key goes through KVStore's index. A secondary lookup goes through the secondary index the fold maintains. A range query is a KVStore iteration over an appropriate key range.

The layer's responsibility during fold is to update its structural state in a way that remains consistent: schema state and index state move together with primary data, all within a single state machine step.

See [`fold.md`](fold.md) for the full treatment of how state is constructed.

## Contract and invariants

- **Structural state is a function of the log.** Two peers with the same linearized log produce identical relational state, including identical indices.
- **No relational state bypasses KVStore.** Every persistent structural artifact (index entries, schema descriptors, table metadata) is stored through the KV layer and thus durable through the PageStore boundary.
- **Indices and primary data stay consistent within a step.** Because the state machine executes one operation at a time, and a single operation updates primary data and all affected indices together, no read ever observes a state where the primary has been updated but an index has not (or vice versa).
- **Schema is reconstructable.** On restart or when a peer catches up on the log, the schema at any point is the one the fold has derived at that log position.

## Open questions

- **Compile-time schema enforcement.** Whether schema constraints can be encoded into the Rust type system so that invalid operations are unrepresentable at compile time — eliminating runtime checks on the hot path — is an open design question.
- **Index configuration.** How secondary indices are declared. Options include schema-level declarations (the schema itself names the indices), operation-initiated declarations (an operation in the log creates a new index), or both.
- **Index backfill.** When a new index is declared, how existing data is brought into it. Under the fold-constructs-state model, this is conceptually a replay of the log through a fold that also maintains the new index — but what that looks like operationally is open.
- **Composite and functional indices.** Whether indices over multiple fields or over computed expressions are first-class.
- **Secondary index invalidation under schema change.** When a schema change affects a field that has a secondary index, what the index represents during and after the change is a design decision.
- **Relationship to query planning.** Whether RelationalStore exposes primitives that a query planner can compose (index scans, joins) or whether planning is entirely above the trait.
- **Cross-table references.** Whether foreign-key-like relationships are recognized by the relational layer (so the fold can encode referential integrity in its construction) or handled entirely in the application layer.

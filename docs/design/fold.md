# Fold

The fold is the transformation that turns a linearized operation log into read-optimized state. It runs over the sequence produced by CRDTStore and builds the materialized state that the layers above (RelationalStore, KVStore, ultimately the application) query.

This page establishes the load-bearing properties of the fold and describes how those properties compose into the architecture's read model and persistence story. Precise trait shape and mechanics for the fold are pending specification; this page describes what is fixed.

## What the fold does

The fold processes the linearized operation log one event at a time and builds a read-optimized data structure. The linearized log is write-optimized — append-only, ordered by the sync protocol's total order, sequential. The constructed state is read-optimized — indexed, structured, queryable. The fold is the bridge.

Reads do not touch the log. Reads query the state the fold constructed. A point lookup goes through the KVStore's index, which the fold maintains. A range scan iterates the KVStore's sorted data. A secondary-index lookup hits the RelationalStore's indices, which the fold built.

This is the single most easily misread aspect of the architecture. The fold is a **write-path operation that produces read-optimized state.** It is not a query mechanism. When the spec says "reads go to the fold-constructed state," it means reads query the materialized output of the fold, not that reads execute the fold.

## Persistence and restart

The fold writes into the lower layers of the stack synchronously. When a fold step completes, its effect on constructed state is already durable through `PageStore`. On restart, the stack reconstructs the upper layers from the durable state `PageStore` holds; there is no separate checkpoint mechanism and no log-replay on the recovery path.

A fresh peer joining a cluster does have to fold from some starting point — it catches up on events through the sync protocol and folds them as they arrive. That is sync's concern, not a recovery mechanism (see [`sync.md`](sync.md)).

Log compaction, undo/redo tail access of the log, and related concerns remain open design questions. They are separate from the restart path.

## Cross-peer invariants are encoded in the fold

Invariants that span peers — uniqueness, referential integrity — are encoded in how the fold is constructed. The fold processes each event in the order sync produced, updating state deterministically; because every peer folds the same sequence identically, every peer produces the same state. Invariants follow from the fold's construction rather than from cross-peer coordination.

A well-constructed fold encodes its invariants without explicit predicates. Users who add predicate gates — functions that examine a candidate event and current state and decide whether to apply — are responsible for correctness; the architecture does not validate them. This is a deliberate narrowing: the choice simplifies the core and narrows its focus away from problem spaces where cross-peer predicate gating is load-bearing.

## Fold composition

Folds compose by concatenation. Two independent fold steps over the same event type concatenate into a single fold; the mental model is that the lower layers of the stack are themselves the accumulator the fold builds into.

Application code that wants to derive its own state from events extends the fold by composing into it. Because the fold is synchronous and deterministic, composition inherits those properties: the same events produce the same state at every peer without additional coordination. There is no architectural distinction between a "primary" fold and "user" folds — extending the fold is composition.

The mechanism by which fold steps are composed (in-process Rust trait composition, other techniques) is an open design question.

## Relationship to the layers

- **CRDTStore** produces the linearized sequence the fold consumes. The fold is downstream of linearization and event application.
- **RelationalStore** is one of the things the fold builds — the fold maintains the secondary indices and schema state that RelationalStore exposes.
- **KVStore** is the physical backing for most of what the fold constructs. The fold writes to it; reads query it.
- **PageStore** is invisible to the fold directly. The fold operates in terms of the KV layer; durability is handled through the layers beneath.

## Contract and invariants

- **Fold output is a function of the log prefix.** Given the same linearized log up to some position, every peer that runs the same fold produces the same constructed state. No source of nondeterminism participates.
- **Reads query constructed state, not the log.** No architectural read path scans the operation log.
- **Fold steps are pure.** Each step takes state and input and produces new state (and optionally output). No hidden dependency on external services, clocks, or randomness.
- **State is durable when the step returns.** A fold step that completes has already committed its effect through the layers beneath; no separate checkpoint step is needed.

## Open questions

- **Exact fold trait shape and mechanics.** Pending specification. What the fold method's signature is, whether it is expressed per-event-type or per-state-type, and related design decisions.
- **Log compaction and undo/redo tail access.** How the log is compacted over time, and whether the stack exposes tail-access semantics for undo/redo patterns, remain entangled design questions.
- **Partial folds and incremental materialization.** For folds whose output is large, whether parts of the constructed state can be lazily materialized on first read rather than built eagerly.
- **Fold composition mechanism.** How fold steps are composed — in-process Rust trait composition, other techniques — is open.

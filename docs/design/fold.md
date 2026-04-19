# Fold

The fold is the transformation that turns a linearized operation log into read-optimized state. It runs over the sequence produced by CRDTStore and builds the materialized state that the layers above (RelationalStore, KVStore, ultimately the application) query.

This page establishes the load-bearing properties of the fold and describes how those properties compose into the architecture's read model, caching approach, and predicate-evaluation story. Precise trait shape and mechanics for the fold are pending specification; this page describes what is fixed.

## What the fold does

The fold processes the linearized operation log one event at a time and builds a read-optimized data structure. The linearized log is write-optimized — append-only, ordered by the sync protocol's total order, sequential. The constructed state is read-optimized — indexed, structured, queryable. The fold is the bridge.

Reads do not touch the log. Reads query the state the fold constructed. A point lookup goes through the KVStore's index, which the fold maintains. A range scan iterates the KVStore's sorted data. A secondary-index lookup hits the RelationalStore's indices, which the fold built. None of these operations scan the operation log.

This is the single most easily misread aspect of the architecture. The fold is a **write-path operation that produces read-optimized state.** It is not a query mechanism. When the spec says "reads go to the fold-constructed state," it means reads query the materialized output of the fold, not that reads execute the fold.

## Fold caching and recovery

Replaying the entire log on every read or restart would be absurd for any long-lived deployment. What the stack does beyond the baseline to avoid this is an open design question. The baseline is committed; specific optimizations are not.

### Pure replay is the baseline

On restart or on a fresh peer, fold the log from its start. The constructed state is whatever the fold produces at the end. Reads then query that state. Cost is proportional to the total log size.

This is correct but unscalable for any long-lived deployment. It is the fallback, not the common path.

### Possible optimizations, pending design

Beyond the baseline, several optimizations are natural extensions of the fold-constructs-state model. None is committed in this spec.

- **Checkpoint-based recovery.** Persist the constructed state at some well-defined position so that restart replays only operations since that position, not from the log's start. The mechanics — what constitutes a checkpoint, how it is advanced, what ensures peers have reached it, what happens when peers fall behind — interplay with log compaction, with peer-list upkeep, and with whether the stack supports reads from the tail of the log for undo/redo. These questions benefit from being settled together rather than independently; the spec does not commit to a design ahead of that work.
- **Invertibility as an optional event extension.** Some events are naturally invertible — applying the event has an opposite that undoes its effect. Where events support inversion and the fold supports it, the fold can rework its state around out-of-order insertions by inverting backward and re-folding forward. Whether this is worth supporting, and at what scope, depends on how checkpoint-based recovery ends up being designed.

Both are possible refinements. Whether the spec eventually commits to either, both, or neither is open.

## Cross-peer invariants as predicates in the fold

Invariants that span peers — uniqueness, non-negative balances, referential integrity — are evaluated as the fold processes events, because that is where the state against which the invariant applies actually exists.

The fold is a sequence of step functions: given a state and the next event, produce the next state. A predicate is a function of the same type: given the state and the candidate event, decide whether to apply it. Because the sequence is deterministic and every peer folds it identically, every peer reaches the same accept-or-skip decision for every event. Convergence is preserved; the invariant is enforced globally; no coordination beyond the sync that was already happening is required.

Skipped events propagate to the application as a distinguishable outcome — the event was accepted locally, but the totally-ordered fold rejected it. Applications that care distinguish between "pending" and "committed" state in their UI to reflect this.

## Fold as user-extensibility surface

Application logic sits at the top of the stack (see the layer diagram in [`overview.md`](overview.md)) — the endpoint where user code consumes the structured state the layers below construct. The fold is where that consumption attaches to the stack's own machinery. Application-shaped state is what the fold produces; extending the fold is how user code shapes that state for its own purposes.

A user-defined fold reads the linearized event stream the core produces and constructs whatever read-optimized state the application wants. Because the sequence is deterministic and each step is pure, a user-defined fold inherits the same properties the primary fold has: same inputs at every peer produce the same output; no coordination required beyond sync.

Specific integration mechanisms for user-defined folds — how user code is loaded, run, and isolated relative to the core engine — are open design questions. The spec does not commit to any particular mechanism as a settled path for first implementations.

## Folds beyond the primary

A single linearized log admits multiple fold consumers. The primary fold constructs the state that behaves as "the database" — KV indices, secondary relational indices, schema-aware structural metadata. Additional folds can construct other read-optimized views of the same data: aggregations, projections, time-travel structures, and so on.

This is a user concern rather than a core architectural promise. The core stack provides the linearized log and the primary fold; additional folds are application- or runtime-layer constructions that consume the log the core produces. They are not a first-class commitment of this spec and have no influence on the trait contracts of the layers below.

## Relationship to the layers

- **CRDTStore** produces the linearized sequence the fold consumes. The fold is downstream of linearization and event application.
- **RelationalStore** is one of the things the fold builds — the fold maintains the secondary indices and schema state that RelationalStore exposes.
- **KVStore** is the physical backing for most of what the fold constructs. The fold writes to it; reads query it.
- **PageStore** is invisible to the fold directly. The fold operates in terms of the KV layer; durability is handled through the layers beneath.

## Contract and invariants

- **Fold output is a function of the log prefix.** Given the same linearized log up to some position, every peer that runs the same fold produces the same constructed state. No source of nondeterminism participates.
- **Reads query constructed state, not the log.** No read path scans the operation log; that would defeat the purpose of the fold.
- **Fold steps are pure.** Each step takes state and input and produces new state (and optionally output). No hidden dependency on external services, clocks, or randomness.

## Open questions

- **Exact fold trait shape and mechanics.** Pending specification. What the fold method's signature is, whether it is expressed per-event-type or per-state-type, and related design decisions.
- **Checkpoint-based recovery.** Whether the stack supports it, what constitutes a checkpoint, when one advances, how the system ensures restart can replay from it. One possible optimization on top of the pure-replay baseline, entangled with compaction and with whether the stack supports undo/redo reads from the log's tail.
- **Predicate rejection representation.** When the fold skips an event because a predicate rejects it, what remains visible in the log is an implementation question with observability implications.
- **Partial folds and incremental materialization.** For folds whose output is large, whether parts of the constructed state can be lazily materialized on first read rather than built eagerly.
- **Additional fold types.** Whether the stack exposes a first-class way for application code or runtime code to register additional folds, or whether all such folds live outside the core stack entirely.
- **Integration mechanisms for user-defined folds.** Whatever the answer to the preceding question turns out to be, the mechanism by which user code implements a fold — in-process using the Rust trait directly, out-of-process consumers communicating over a protocol, sandboxed module formats, embedded script runtimes — is a separate open question with several valid answers. No mechanism is settled.

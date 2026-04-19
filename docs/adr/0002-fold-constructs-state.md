- **Status:** Accepted
- **Date:** 2026-04-18
- **Supersedes:** _(none)_
- **Superseded by:** _(none)_

# Fold constructs state; reads query constructed state

## Context

Depends on [ADR 0001](0001-linearize-during-sync.md). With operations arriving in a deterministic total order from the sync boundary, the architecture needs a mechanism that turns the event stream into read-optimized state.

The layer stack puts reads through the KVStore and RelationalStore layers (see [`../design/overview.md`](../design/overview.md)). Those layers expose indexed, structured, queryable state, not an operation log. The fold is the transformation that produces that state.

## Considered options

- **Fold as a write-path operation that constructs read-optimized state.** The fold processes the linearized log event by event on the write path and builds the materialized state the layers above expose. Reads query that state directly.

This is the only framing that was weighed during architecture design. Alternative framings — fold as a read-time query mechanism, or fold as a separate asynchronous aggregation layer — surfaced during subsequent exposition and are worth distinguishing from (see the Decision section), but were not considered alternatives at the design stage.

## Decision

The fold is a write-path transformation that constructs read-optimized state. Reads query that state.

Two potential misreadings are worth distinguishing from:

- The fold is not a read-time query mechanism. Reads do not scan the log; reads query the state the fold built.
- The fold is not a separate asynchronous aggregation layer. It runs synchronously in the write path as part of the state machine; it is not a downstream consumer.

Precise trait shape and mechanics for the fold are pending specification and tracked in [`../design/fold.md`](../design/fold.md). This ADR fixes the direction: fold is construction, not query.

## Consequences

- **Reads go through the KVStore and RelationalStore layers.** Point lookups are KV lookups. Range scans are KV iteration. Secondary-index queries are RelationalStore lookups. Those layers maintain the state the fold built. The event log is not hidden from users who want to consume it directly for their own purposes; the architecturally privileged read path is through the constructed state.
- **Invariants are encoded in the fold's construction.** Because operations arrive pre-ordered (per ADR 0001) and the fold is deterministic, every peer produces the same state. Invariants that span peers are encoded by how the fold is constructed, not by separate predicate-based rejection. Well-constructed folds encode their invariants without explicit predicates. Users who add predicate gates are responsible for their correctness — the architecture does not validate them. This is a deliberate narrowing: the choice simplifies the architecture and narrows its focus away from problem spaces (like financial settlement) where cross-peer predicate gating is load-bearing.
- **Folds compose by concatenation.** Two independent fold steps over the same event type concatenate into a single fold; the mental model is that the lower layers of the stack are themselves the accumulator the fold builds into. There is no architectural distinction between a "primary" fold and "user" folds — extending the fold is composition.
- **State persists synchronously through the stack.** When a fold step completes, its effect on constructed state is already durable through `PageStore`. Restart reconstructs the upper layers from the durable state `PageStore` holds; there is no separate checkpoint mechanism and no log-replay on the recovery path.
- **The architectural role of RelationalStore becomes sharper.** RelationalStore's contribution is the indexed state it exposes. That state is built by the fold together with the CRDT layer's metadata. The layer is not a query engine with an execution planner; it is a structured view of constructed state.

## Open questions

- The precise trait shape for the fold — signature, per-event-type versus per-state-type expression, return value, state-reference lifetimes — is pending specification in [`../design/fold.md`](../design/fold.md).
- The mechanism by which fold steps are composed (in-process Rust trait composition, other techniques) is open.

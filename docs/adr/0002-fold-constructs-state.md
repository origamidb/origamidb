- **Status:** Accepted
- **Date:** 2026-04-18
- **Supersedes:** _(none)_
- **Superseded by:** _(none)_

# Fold constructs state; reads query constructed state

## Context

Depends on [ADR 0001](0001-linearize-during-sync.md). With operations arriving in a deterministic total order from the sync boundary, the natural follow-on question is how state is produced for reads. Several framings were considered during the design discussion; one was specifically flagged by the designer as a misread of the architecture.

The layer stack puts reads through the KVStore and RelationalStore layers (see [`../design/overview.md`](../design/overview.md)). Those layers expose indexed, structured, queryable state — not an operation log. The question is where that state comes from and when it is built.

## Considered options

- **Fold as a read-time query mechanism.** Scan the linearized operation log at query time and compute the answer. This framing appeared during early design discussion and was corrected: it puts every query on a linear scan path, negates the point of indexed KV layers, and makes read performance proportional to log size. The architecture already committed to cache-friendly reads; a read-time fold is incompatible with that commitment.
- **Fold as a separate aggregation layer above the read path.** Materialized views computed asynchronously from the log, similar to downstream consumers of a streaming system. Partial fit — it produces queryable state — but leaves unexplained why the core stack bothers with the CRDTStore and RelationalStore layers at all, and why the event-fold contract is the shape it is. A separate asynchronous aggregation layer is something some applications might build on top; it is not what the core stack runs.
- **Fold is a write-path operation that constructs read-optimized state (chosen).** The fold processes the linearized log event by event on the write path and builds the materialized state the layers above expose. Reads do not touch the log; reads query the state the fold constructed.

## Decision

The fold is a write-path transformation that constructs read-optimized state. Reads query that state, not the log.

Precise trait shape and mechanics for the fold are pending specification and tracked in [`../design/fold.md`](../design/fold.md). This ADR fixes the direction: fold is construction, not query.

## Consequences

- **Reads go through the KVStore and RelationalStore layers, not through the log.** Point lookups are KV lookups. Range scans are KV iteration. Secondary-index queries are RelationalStore lookups. Those layers maintain the state the fold built; the log is a write-path artifact.
- **Invariants become fold predicates, made concrete.** Invariants that traditional systems would enforce with cross-peer coordination become predicates the fold evaluates against the linearized log. Because the log is deterministic (ADR 0001), every peer makes the same accept-or-skip decision. This consequence follows from ADR 0001; the present decision is what makes it concrete — the predicate runs in the fold because that is where the state the predicate checks against actually exists. See [`../design/fold.md`](../design/fold.md).
- **The decision creates surface for optimization strategies.** Because the fold builds state incrementally, checkpoints and invertibility-based rework become natural extensions of the model — but their shape is a later design question, entangled with log compaction and peer-list upkeep (see [`../design/fold.md`](../design/fold.md)). This ADR fixes the construction direction (write-path fold builds read state); it does not fix any specific recovery optimization.
- **User-defined folds become possible in the same shape.** Because the primary fold is write-path construction over a linearized stream, additional folds of the same shape can consume the same stream and construct their own read-optimized state. This is user-space, not a first-class architectural promise; the cross-reference in [`../design/fold.md`](../design/fold.md) names it explicitly.
- **The architectural role of RelationalStore becomes sharper.** RelationalStore's contribution is the indexed state it exposes, built by the fold. The layer is not a query engine with an execution planner; it is a structured view of what the fold constructed.

## Open questions

- The precise trait shape for the fold — signature, per-event-type versus per-state-type expression, return value, state-reference lifetimes — is pending specification in [`../design/fold.md`](../design/fold.md).
- Checkpoint mechanics (what constitutes a checkpoint, when and how it advances, how it is persisted) are pending. See [`../design/fold.md`](../design/fold.md).
- Predicate rejection representation and the user-extensibility mechanism for additional folds are both tracked in [`../design/fold.md`](../design/fold.md)'s open questions.

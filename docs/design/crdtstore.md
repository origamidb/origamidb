# CRDTStore

CRDTStore is the convergence layer. It is where concurrent operations from peers become a single, deterministic, totally ordered sequence that the fold above can apply step by step. It is the layer that makes OrigamiDB a convergent distributed system rather than a single-writer one.

The layer is generic over a foldable event trait — in OrigamiDB's design, `fold` lives on the event type itself. Different applications define different event types with different fold behaviors (last-writer-wins, additive counters, sequence algorithms) without affecting any other layer. The CRDTStore trait constrains the shape of the surrounding contract, not the fold strategy of any individual event.

## Responsibilities

A CRDTStore implementation:

- Produces a deterministic total order over concurrent operations from peers, using the sync protocol described in [`sync.md`](sync.md).
- Applies each event to state step by step against that totally ordered sequence, by invoking the event's own fold behavior.
- Coordinates with the layer above — Partitioning — for inbound and outbound sync traffic, but does not perform IO itself.

The essential property the layer preserves is convergence: any two peers that have received the same set of operations produce the same constructed state, in any order they received them.

## What CRDTStore does not do

The layer's absences are load-bearing consequences of linearize-during-sync. Several things that traditional CRDT implementations carry are deliberately not present here.

- **No vector clocks or causal metadata inside the foldable event trait's inputs.** By the time an event reaches the fold, its position is already decided. Causal context, if needed, belongs to the sync protocol's ordering or to the event type itself — not to `fold`.
- **No tombstones managed inside the foldable event trait.** Traditional op-based CRDTs carry tombstones to handle deletion after concurrent insertion. Linearized ordering makes tombstone tracking a sync-protocol and state-layer concern, not a property of the trait on the event.
- **No commutativity requirement on individual events.** Two events that would need to commute in a traditional CRDT may or may not here — the linearized order decided which one comes first. `fold` is not a merge in the algebraic sense.
- **No validation of whether an event should be applied.** Whether an event violates an invariant or a prior event supersedes it — those concerns are handled by how the fold above is constructed (see [`fold.md`](fold.md)), not by the event's own `fold`. The event's `fold` assumes the caller decided to apply it.
- **No sync or linearization logic.** Producing the total order is the sync protocol's job, described in [`sync.md`](sync.md). CRDTStore consumes that order and drives the fold. The two concerns are adjacent, not the same.

## Linearize-during-sync

See [`overview.md`](overview.md) for the cross-cutting description and [`../adr/0001-linearize-during-sync.md`](../adr/0001-linearize-during-sync.md) for the rationale. The CRDTStore-specific consequence: by the time an event reaches the fold, its position relative to every other event is already decided. Individual events do not need to be commutative — they arrive pre-ordered, and `fold` applies them in that order.

## Foldable event trait

The trait is implemented on the event type itself. The shape, currently understood approximately:

```text
trait FoldableEvent {
    type State;

    fn fold(&self, state: &mut Self::State);
}
```

Each event knows how to apply itself to a state. CRDTStore walks the linearized sequence and invokes each event's fold, updating state as it goes. No per-event metadata about other peers, no reconciliation logic, no commutativity proof.

The exact signature is still being refined; the shape is committed. Here "shape" means the skeleton — an event folds into a state — while "exact signature" covers details like lifetime parameters on state references, whether `fold` returns anything, whether `State` is an associated type or a generic parameter, and similar trait-design decisions.

Several properties follow from this shape:

- **Operations need not commute.** Events arrive already ordered by sync. Two events that would need to commute in a traditional CRDT may or may not here — it does not matter, because sync decided which one comes first.
- **No per-event causal metadata.** Causal context, if needed, is a property of the sync protocol's ordering or of the event type itself, not an input to `fold`.
- **Invariant-encoding is not part of fold-on-event.** How an invariant is enforced — by construction in the composed fold above, or by user-added predicates on top — is a fold-level concern (see [`fold.md`](fold.md)). The event's `fold` assumes it is one the caller is applying.

### Composition of event types

A typical domain has values of different kinds: scalars, counters, sequences, sets. Each kind wants a different fold behavior. This is expressed as composition over the foldable event trait, not inside it.

A compound state is a structure whose fields are each affected by events with different fold behavior. Events are typed so that each dispatches to the part of state it affects. The dispatch is a standard Rust match or enum, monomorphized at compile time. There is no dynamic resolution at runtime; the compiler resolves every fold call to a concrete type.

This is how last-writer-wins registers, additive counters, and sequence algorithms coexist in the same CRDTStore without the trait itself knowing about any of them.

### Optional invertibility

For some use cases (log compaction, undo/redo tail access — see [`fold.md`](fold.md)), it is useful to be able to invert an event — apply its opposite to undo its effect on state. Where events are naturally invertible (a counter increment can be inverted to a decrement; an LWW write can be inverted back to its prior value if that value is known), this would become an optional extension to the foldable event trait rather than a core requirement. Whether the stack commits to this extension depends on how those broader design questions settle; it is open.

## Semantic priority is a property of the event type

The sync ordering function consults a semantic-priority property exposed by the event type itself (see [`sync.md`](sync.md) for the function's shape). That property is not part of CRDTStore's trait contract with events; it rides on a trait separate from the foldable-event contract. The fold on the event, operating downstream of sync, does not consult semantic priority — by the time `fold` runs, semantic ordering has already been baked into the sequence.

This separation keeps the foldable event trait simple and keeps the sync protocol the single place where "what order should these events be in" is resolved.

## Relationship to other layers

- **Downward to RelationalStore.** Once a sequence has been linearized and fold has produced the next state, CRDTStore is transparent. The constructed state flows through to RelationalStore, which exposes primary data, secondary indices, and schema as a structured view.
- **Upward to Partitioning.** CRDTStore does not open sockets. It produces outbound intents — sync messages to send to named peers — and receives inbound messages as state machine inputs. Partitioning (and the security layer at its boundary) handles transport. See [`partitioning.md`](partitioning.md).
- **Sideward to the fold.** The fold is not a separate layer; it is the read-path construction logic that runs over the linearized sequence CRDTStore produces. See [`fold.md`](fold.md).

## Cross-peer invariants

Invariants that span peers — uniqueness, referential integrity — are not enforced inside CRDTStore. They are encoded in how the fold above is constructed. Because the linearized sequence is deterministic and all peers fold it identically, all peers produce the same state, and invariants follow from the fold's construction rather than from cross-peer coordination.

Linearization is the coordination mechanism. It eliminates the need for reservations, compensations, or consensus protocols as separate systems — those exist in traditional CRDT systems precisely because partial orders make per-peer decisions inconsistent. With a total order, the fold's state evolves identically at every peer.

The fold is where invariants live because that is where state is constructed. See [`fold.md`](fold.md).

## Contract and invariants

- **Deterministic fold over a total order.** Given the same linearized sequence and the same event types, every peer produces the same state.
- **No per-event reconciliation.** The trait does not carry vector clocks, tombstones, or causal metadata as inputs to `fold`. Those concerns are absorbed by sync.
- **Generic over a foldable event trait.** The CRDTStore trait is parameterized at compile time. Different stacks can compose different event types without any other layer changing.
- **Synchronous with the rest of the stack.** CRDTStore is a state machine with the same interface shape as every other layer. It does not perform IO; it returns outbound sync intents as outputs.

## Open questions

- **Exact shape of the foldable event trait.** The skeleton — an event folds into a state — is the committed form. Associated types versus generic parameters, lifetime parameters on state references, whether `fold` has a return value, and similar refinements are trait-design decisions that affect ergonomics but not the core contract.
- **Semantic priority mechanism.** How each event exposes its semantic priority — a numeric field, a trait method, a derive macro over an enum — is unresolved. The choice affects how application code defines new event types.
- **Compound event decomposition.** A single input to the state machine may represent a logically compound event that internally affects multiple fields or multiple domain objects. Whether the fold sees compound events as atomic units or as decomposed sub-events is a trait-design decision.
- **Invertibility contract.** For implementations that support inversion, the exact shape of the invert operation (returns an inverse event, applies the inverse in place, produces a diff) is undecided.

- **Status:** Accepted
- **Date:** 2026-04-18
- **Supersedes:** _(none)_
- **Superseded by:** _(none)_

# Linearize during sync

## Context

OrigamiDB is a distributed system in which peers must converge on a single consistent state after exchanging events. Distributed systems that need replicas to agree on state traditionally take one of two paths.

**Consensus-based systems** (Raft, Paxos) coordinate across replicas before an operation is considered committed. A write is not durable or visible until a quorum of replicas has agreed on its position in the order. This pays coordination cost on every operation and is partition-intolerant by design.

**Traditional CRDT systems** take the opposite approach. Each peer accepts writes immediately without coordination. The system relies on properties of the operations (commutativity, associativity, idempotence) and per-operation metadata (vector clocks, tombstones, causal context) to guarantee that any two peers that have received the same set of operations arrive at the same state regardless of the order those operations arrive. Coordination cost is zero at write time; ordering cost is paid on every merge or read.

A recent research direction (Eg-walker and similar) records simple indices per operation and replays relevant causal history on demand when the system needs to resolve a concurrent edit. This reduces per-operation metadata while still pushing ordering cost toward read or merge.

OrigamiDB's design has already committed to an embeddable-plus-distributed stack with `no_std`-friendly lower layers and a single synchronous state machine per peer (see [`../design/overview.md`](../design/overview.md) and [`../../ARCHITECTURE.md`](../../ARCHITECTURE.md)). Per-operation metadata on every write would enlarge the hot-path state in ways that work against the allocation-free embedded target. Per-read replay of causal history would put expensive work on what should be a cache-friendly query path. Global consensus on every write is incompatible with partition tolerance and with the embeddable deployment topology.

A third path is available: coordinate neither on write nor on read, but on sync.

## Considered options

- **Consensus-based pre-ordering (Raft/Paxos).** Coordination cost on every write. Partition-intolerant. Quorum requirements make embedded and offline deployment topologies awkward at best. Strong ordering guarantees, but at a cost the architecture does not want to pay on the hot path.
- **Traditional CmRDT/CvRDT reconciliation at merge time.** Requires commutativity and associativity of operations plus per-operation metadata (vector clocks, tombstones). Per-operation state grows with the system's history; tombstone management is a known source of complexity in real implementations.
- **Eg-walker-style replay at read time.** Records simple indices per operation and replays causal history when a concurrent edit needs to be resolved. Reduces per-operation state at the cost of putting replay work on the read path, which fights the cache-friendly read model the rest of this architecture is designed for.
- **Linearize during sync (chosen).** Peers accept writes without coordination; the sync protocol produces a deterministic total order over all concurrent operations at the sync boundary. Downstream of sync, the fold runs over an already-totally-ordered log.

## Decision

OrigamiDB linearizes concurrent operations into a deterministic total order at the sync boundary. Every peer runs the same deterministic ordering function over the same set of operations and produces the same totally ordered sequence.

The mechanism lives in [`../design/sync.md`](../design/sync.md). This ADR fixes the choice, not the mechanism.

## Consequences

- **The foldable event trait stays small.** Because operations arrive pre-ordered, individual events do not need to be commutative. `fold` lives on the event type itself and applies an event to state without needing to reason about what other peers did. See [`../design/crdtstore.md`](../design/crdtstore.md).
- **Cross-peer invariants become predicates in the fold.** Invariants that span peers (uniqueness, non-negative balances, referential integrity) are evaluated by the fold against the totally ordered log. Every peer makes the same accept-or-skip decision because the log is deterministic. Reservations and compensation patterns from traditional CRDT systems are not needed as separate subsystems; the coordination already happened at sync. See [`../design/fold.md`](../design/fold.md).
- **Per-operation metadata stays small.** Vector clocks, tombstones, and causal context are not inputs to `fold`. The hot path does not pay for ordering machinery that sync has already handled.
- **Sync carries the ordering cost.** The work does not vanish — it moves to a specific architectural boundary. Sync-time bandwidth, the correctness of the comparison function, and the reliability of the delta-exchange protocol become load-bearing concerns. The spec addresses these in [`../design/sync.md`](../design/sync.md).
- **Partition tolerance is preserved.** Because peers accept writes without coordination, each peer remains available under partition. Convergence is postponed until sync is possible, not blocked entirely.

## Open questions

- The detailed mechanics of the ordering function, the causal-DAG structure, and the delta-exchange protocol are pending specification in [`../design/sync.md`](../design/sync.md).
- The exact shape of the foldable event trait is pending specification in [`../design/crdtstore.md`](../design/crdtstore.md).
- Specific mechanisms for checkpoint-based recovery and log compaction relative to linearization are pending and tracked in [`../design/fold.md`](../design/fold.md) and [`../design/sync.md`](../design/sync.md).

# Sync

Sync is the protocol by which peers exchange events and converge on a single, totally ordered log. It is the work that makes every other design decision in the architecture possible: once sync has produced a deterministic total order, the fold simplifies to a sequential step-per-event, cross-peer invariants become per-step predicates, and the fold's trait contract stays small.

The specifics of the sync protocol — how the causal DAG is structured, what the ordering function operates on, how peers negotiate what to exchange — are pending detailed specification. This page describes the shape the protocol takes and the properties it must satisfy.

## The shape

Each peer keeps its locally accepted events in a structure that supports ordered exchange with other peers. When two peers sync, they exchange the events since some shared confirmed position, and each peer independently applies the same deterministic ordering function to produce the same merged sequence.

Because the ordering function is deterministic, total, and identical at every peer, any two peers that have received the same set of events produce the same totally-ordered log.

## Structural resemblance to LSM compaction

The protocol is structurally the merge step of merge sort, applied to distributed event logs rather than on-disk sorted runs. The shape is the same one used in log-structured merge trees: each peer's local events are one sorted run; syncing two peers is the compaction that merges two sorted runs into a larger sorted run; the total order across the stack is the long-run invariant the merging preserves.

The resemblance is not incidental. The same mechanics — sorted buffers, merge as a single linear pass, amortized ordering cost — are the reason both patterns are efficient. Implementation work at one layer can inform the other, and an implementation may be able to share code across the two processes if the trait abstractions align.

The resemblance does not mean the two processes are the same. An LSM compaction at the KVStore layer is an operation on an individual peer's stored pages. A sync between two peers is a cross-peer exchange that advances the linearized log. They share shape; they do not share purpose.

## The ordering function

Sync produces a deterministic total order. The ordering uses two layers of comparison:

1. **Semantic priority.** The type or intent of an event influences its position relative to others.
2. **Mechanical fields** (peer identifier, logical timestamp, content hash, or similar) as the final tiebreaker so that no two events ever compare equal.

What is fixed about the ordering function: it is deterministic, total, and identical at every peer.

## Discovery of the delta

To exchange "events since we last agreed on state," both peers need to know where that "last agreed on state" is. That position is a shared confirmed point reflecting events both sides have incorporated. Each peer tracks its own view of this position for every peer it syncs with.

The specifics of how the confirmed point is maintained, advanced, and bounded are pending specification. What is fixed: sync exchange is a delta over events since the confirmed point, not a full-log exchange.

## Properties the protocol must satisfy

Regardless of the specific mechanism chosen, the sync protocol must produce:

- **Deterministic output.** Given the same set of events at two peers and the same ordering function, sync produces the same totally ordered sequence at both peers.
- **No network IO inside the state machine.** Sync computes what to send and what to do with what was received. Transport is returned intent.
- **Idempotent under duplication.** An event received more than once (from different sync paths, from retry) produces no additional effect beyond its first application in order.
- **Bounded delta work.** The cost of a sync is a function of the delta size, not of the total log size.

## Sync is state machine IO

The sync protocol is implemented as a state machine at the Partitioning layer. Inbound sync messages arrive as inputs. Outbound sync messages leave as outputs, returned to the caller as intents rather than performed directly.

The security layer sits at the IO boundary between Partitioning and the network. Inbound bytes are verified and decrypted before reaching the sync state machine. Outbound sync intents are signed and encrypted before transmission. Sync itself operates only on authenticated, verified input — it does not concern itself with transport security.

See [`partitioning.md`](partitioning.md).

## Optional bandwidth optimization

For very large deltas — catching up after a long offline period, bulk joining a cluster — transmitting the whole delta may be wasteful if both peers already have most of the events. Sophisticated set reconciliation primitives (rateless or structured probabilistic filters, recursive range fingerprinting) can identify the exact symmetric difference with bandwidth proportional to the diff, not the delta.

These are optimizations layered over the core protocol. The architectural commitment is to deterministic sorted-merge semantics as the underlying mechanism; any bandwidth-reducing optimization that preserves that invariant is compatible. The spec does not currently commit to any specific optimization.

## Open questions

- **Exact DAG and ordering mechanics.** What the causal DAG looks like, what the ordering function operates on, how it is represented — pending specification.
- **Confirmed-position mechanics.** What constitutes a shared confirmed point, how it advances, how it is persisted and coordinated.
- **Streaming vs batch exchange.** Whether events since the confirmed point are transmitted as one batch per sync, or streamed continuously.
- **Retry and idempotence mechanics.** How a sync that is interrupted is resumed, and how idempotence is guaranteed at the implementation level.
- **Multi-peer topology.** Whether sync is strictly pairwise with topology decisions made at the Partitioning layer, or whether the sync protocol itself describes multi-peer patterns.
- **Optional bandwidth optimization.** If and when a specific set-reconciliation primitive is adopted, how it composes with the core merge.
- **Event identity.** Whether every event carries a content hash, a composite `(peer, sequence)` identifier, or some other unique address, and how that identity participates in the ordering function's final tiebreaker.

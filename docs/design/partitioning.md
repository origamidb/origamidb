# Partitioning

Partitioning is the top layer of the core trait stack. It owns sharding, replication, and cluster membership, and it is where the network enters the stack — though it does not itself perform network IO. Inbound packets arrive as inputs to the state machine; outbound packets leave as intents for an external runtime to transmit.

The trait name is working. A final name may replace "Partitioning" once the layer's responsibilities stabilize (see the spec-wide open questions in [`overview.md`](overview.md)). What the layer does is clear; what it is called is not final.

## Responsibilities

A Partitioning implementation:

- Holds cluster membership as state that is itself replicated through the CRDT layer beneath it. Every peer's view of who else exists in the cluster converges by the same mechanism that converges application data.
- Computes shard and replica assignments deterministically from the current membership. Because membership is converged, all peers arrive at the same shard map from the same membership state.
- Routes operations to the shard or replica responsible for them. Where an operation belongs to a remote shard, the layer produces an outbound intent rather than applying the operation locally.
- Produces outbound sync intents and consumes inbound sync messages as state machine IO, handing verified events to the CRDTStore layer below.

## Membership as replicated data

Cluster membership — the set of peers, their roles, their sync state, their shard ownership — is stored in the same stack as everything else. It is data in the CRDTStore layer. Adding a peer is an event in the log. Removing a peer is an event in the log. Any property of a peer's role that changes over time is a sequence of events.

This has two consequences. First, there is no external coordination service. No ZooKeeper, no etcd, no separate consensus cluster. The CRDT layer handles convergence on membership exactly as it handles convergence on any other data.

Second, the properties the architecture guarantees for data apply to membership: linearize-during-sync means all peers see the same sequence of membership events in the same order; fold-constructs-state means each peer derives its current view of membership from the log.

### Deterministic shard assignment

Given a converged view of membership, the shard assignment — which peers own which ranges of the keyspace, which peers are replicas for which shards — is computed deterministically. Every peer with the same membership view derives the same shard map.

The specific assignment function is an implementation choice. Consistent hashing, rendezvous hashing, and range partitioning over the primary key each produce valid assignments; each has different properties under peer churn. The spec does not prescribe a choice. What it does prescribe is that the choice be deterministic: given the same membership state, every peer produces the same shard map, without coordination.

## Network IO as state machine input and output

The Partitioning layer does not open sockets. It does not read bytes from the network, and it does not write bytes to it. What it does is model inbound network activity as inputs to its state machine and outbound network activity as outputs the state machine returns.

Inbound: the external runtime (the event loop, the network driver, the transport library — whatever the deployment uses) receives bytes from a peer, hands them to the security layer for verification and decryption, and then passes the verified payload as an input to the Partitioning state machine. The state machine processes it like any other input, possibly producing outputs.

Outbound: when the state machine determines that a peer should receive a message (an outbound sync delta, a membership update, a routed operation), it returns that as an output. The output describes the intended transmission — the target peer, the payload — but does not perform it. The external runtime takes the intent, applies the security layer to sign and encrypt it, and transmits it on whatever transport the deployment uses.

This is the returned-intent pattern the [overview](overview.md) describes, applied at the Partitioning boundary. It keeps the stack pure: the state machine has no hidden IO, no async, no callbacks into runtime services. Everything the stack wants to do in the world is represented as data the caller can inspect, test against, and dispatch.

The pattern is also what makes simulation testing straightforward. A test harness can feed inputs to the state machine, collect outputs, and verify behavior without any actual network. Determinism is a property of the machine, so simulation results are reproducible.

## Security at the IO boundary

Authentication, integrity verification, and encryption live in a layer that sits between Partitioning and the external network. This layer is itself a state machine, composing into the stack by the same generic-over-trait pattern as the layers below.

On the inbound path: raw bytes arrive from the network. The security layer verifies signatures, checks that the peer is a known member of the cluster (membership again, looked up from the converged state), and decrypts payload where applicable. Only verified, authenticated messages enter the Partitioning state machine. Malformed, unauthenticated, or replayed input is dropped at the security boundary and never seen by the rest of the stack.

On the outbound path: the Partitioning layer produces an outbound intent. The security layer signs it and encrypts it for the destination peer. The runtime transmits the resulting bytes.

This concentrates every security concern in one layer with a well-defined surface. Every layer below the security boundary operates on trusted data. The architectural invariant is the same as the trust boundary in a firewall or a trusted execution environment: verify at the boundary, trust everything inside.

The spec does not specify which cryptographic primitives the security layer uses. That is an implementation decision influenced by the deployment — an embedded target may use hardware-accelerated primitives; a server may use a standard library. What the spec does specify is the interface shape: bytes in, verified events out; events in, signed bytes out.

## What Partitioning does not do

- **No transport implementation.** Partitioning does not open TCP connections, send UDP packets, or manage WebSocket sessions. Those are the runtime's responsibility.
- **No consensus protocol.** There is no Raft, no Paxos, no leader election, no quorum agreement at this layer. Convergence on membership and on data flows through the CRDT layer.
- **No query routing by predicate.** Routing is by key: the shard map translates operation keys to peers. Query plans that span shards are an application-layer or coordinator concern, not a Partitioning concern.
- **No durable state of its own outside the CRDT layer.** Everything Partitioning needs to persist — membership, shard configuration, per-peer sync state — is stored through CRDTStore and the layers beneath it.
- **No concurrency inside the layer.** Like every other layer, Partitioning is a synchronous state machine. It processes one input at a time.

## Contract and invariants

- **Membership is convergent.** Any two peers that have received the same set of membership events derive the same membership view.
- **Shard assignment is deterministic.** Given the same membership view, any two peers compute the same shard map.
- **No externally observable mutation here.** Partitioning never opens a connection, reads a byte, writes to medium, or calls a blocking system function. Any durable state changes flow through CRDTStore and the layers beneath. All IO is returned as intent.
- **Verified input only.** Inputs to the Partitioning state machine have already been authenticated, decrypted, and integrity-checked by the security layer.
- **Synchronous contract with the rest of the stack.** One input at a time; outputs returned per step; determinism end to end.

## Open questions

- **Final name for this layer.** "Partitioning" captures part of the role (sharding) but not all of it (membership, network IO framing, replication). A final name is unsettled.
- **Shard assignment algorithm.** Consistent hashing, rendezvous hashing, range partitioning over primary key, or something else. The trade-off is under peer churn: which algorithm moves the least data when a peer joins or leaves.
- **Peer identity and authentication.** How peer identity is established. Options include pre-shared cryptographic identities, certificate chains, capability-based delegation, or a combination. The spec commits to "the security layer verifies inputs"; it does not commit to how peer identities themselves are bootstrapped.
- **Peer departure and eviction.** How a peer that has gone permanently offline is evicted from the cluster. Options include explicit removal events submitted by a remaining peer, timeout-based eviction as part of membership, or quorum-based declarations of unreachability. Each has different availability implications.
- **Cross-shard operations.** Operations that logically span multiple shards — a compound operation touching keys owned by different peers — require a coordination pattern. Whether this is expressed at the Partitioning layer or in a layer above is unresolved.
- **Security layer as its own trait vs. sub-concern.** Whether the security boundary is a distinct trait in its own right or a concern inside Partitioning. See [`overview.md`](overview.md) for the spec-wide position on this.
- **Replication factor and placement policy.** How many replicas each shard has, and on which peers, and whether this is fixed at configuration time or dynamic.
- **Backpressure and flow control.** How the Partitioning layer handles the case where a peer is unreachable long enough that outbound intents accumulate, and whether flow-control signals propagate back into the stack.

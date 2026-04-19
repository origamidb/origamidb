# OrigamiDB design

Per-layer design pages for OrigamiDB. Design intent lives here as architecture develops; once implementations land, code becomes the authoritative behavioral description and this material transitions to design history.

**Status:** Pre-implementation. The per-layer pages are the first authoritative draft of the trait stack.

## How to read this

Read [`overview.md`](overview.md) first. It describes how the layers compose, the properties the stack satisfies, and the cross-cutting concepts that appear at more than one layer. Everything else assumes that foundation.

The per-layer pages describe the contract each trait upholds and the invariants that layer preserves. The cross-cutting pages describe concepts that span multiple layers — the fold that constructs state, the sync protocol between peers.

Each page ends with a section of open questions. These are design decisions that are not yet settled.

## Page shape

Each page roughly follows this rhythm:

1. An orientation paragraph stating what the layer or concept is.
2. Responsibilities — what the layer or concept does.
3. What it does not do — absences that are load-bearing, stated explicitly. Absence-shaped invariants are often the most important thing in design documentation because they are what a new contributor cannot infer from reading code.
4. The substantive content for the page — implementation considerations, composition with adjacent layers, specific contracts or mechanisms.
5. Contract and invariants — a short list of what the layer guarantees and what callers can rely on.
6. Open questions — design decisions not yet settled.

This is a convention, not a template. Pages that need a different structure for a good reason may depart from it. The rhythm is documented because consistency helps readers, not because uniformity is itself a value.

The grounded-vs-inferred discipline described in [`../../AGENTS.md`](../../AGENTS.md) applies: the pages state what is, open questions mark what is not yet settled, and claims not grounded in maintainer guidance should be flagged as inferences or deferred to the open-questions section.

## Contents

- [`overview.md`](overview.md) — layer composition, properties of the stack, and cross-cutting concepts
- [`pagestore.md`](pagestore.md) — raw byte storage; the side-effect boundary
- [`kv.md`](kv.md) — ordered key-value lookup structures over pages
- [`relational.md`](relational.md) — secondary indices and schema over the key-value layer
- [`crdtstore.md`](crdtstore.md) — convergence; generic over a foldable event trait; linearize-during-sync
- [`fold.md`](fold.md) — how the linearized log is transformed into read-optimized state
- [`sync.md`](sync.md) — the inter-peer protocol that produces the linearized total order
- [`partitioning.md`](partitioning.md) — sharding, membership, and networking as state machine IO

## Relationship to other parts of the repo

- [`../../ARCHITECTURE.md`](../../ARCHITECTURE.md) — the short worldview. Names the layers, states the invariants, gives the codemap. The pages here are the deep treatment of what that document summarizes.
- [`../../README.md`](../../README.md) — high-level description of what OrigamiDB is.
- [`../adr/`](../adr/) — architecture decision records. ADRs capture rationale for load-bearing choices; the design pages describe the choices in their resolved form.

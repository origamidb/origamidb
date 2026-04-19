# AGENTS.md

Guidance for LLM-assisted contributors working on OrigamiDB.

OrigamiDB is an embeddable database with a distributed consensus model drawing from ideas used in CRDTs, built in layers of composed Rust traits. The repository is pre-implementation. There is no source code to modify yet beyond a stub `src/lib.rs`. Most contributions at this stage are documentation, RFCs, and the early trait scaffolding.

For the worldview — the layer stack, invariants, and cross-cutting concerns — read `ARCHITECTURE.md`. This file is short and focused on what an agent needs to know to work productively here without violating the design.

## Hard rules

These are non-negotiable architectural invariants. Patches that violate any of them are wrong by construction.

1. **Side effects are confined to `PageStore`.** Higher layers take `&mut self` on their own trait methods and own an implementation of the layer beneath them; they may mutate their own internal state during an operation. What they must not do is perform externally observable mutation — that stays at `PageStore`. Higher layers are deterministic functions over their arguments and the `PageStore` state beneath. This enables stub-free testing via an in-memory `PageStore` and gives effect reasoning in a language without effect types. Do not introduce hidden side effects — syscalls, threads, IO — in layers above `PageStore`.

2. **Trait methods are synchronous.** The shape is `(&mut self, input) -> Output`, where `Output` may be `Option<T>` on a per-method basis — it is not universal. No `async`, no threads, no channels, no blocking inside any layer of the stack. Intended side effects of higher layers bubble up by return and may allow stateful continuations — effectively what `Future::poll` and regenerator-runtime compile into under the hood.

3. **No `Result<T, E>` for soft errors.** If it is not fatal, recover. Restart is a form of recovery, not a fatal condition. Conditions that look like errors are either normal (a method with an `Option` return producing `None` when there is nothing to return) or truly fatal — hardware failure, disk crashes, flash burnout, OS lying about persistence. Do not introduce error variants for partial or recoverable failure.

4. **Operations need not commute.** The convergence layer linearizes concurrent operations during sync. By the time the fold runs, ordering is already decided. Do not reach for vector clocks, commutativity proofs, or tombstone management inside the foldable event trait.

5. **The fold constructs state, it does not read.** Reads query the constructed state, not the operation log. Do not implement read paths that scan the log. Precise trait shape and mechanics are pending specification; this rule is directional, not yet precise.

6. **The core is `no_std`-friendly.** Reach for `alloc` only behind a feature flag. The lower layers should remain allocation-free where the data structure permits. New `PageStore` implementations belong in the core crate behind feature flags, not as separate crates, unless they are genuinely exotic (new platform with significant dependencies).

## Anti-patterns to flag, not to apply

When a task seems to require any of these, stop and open a GitHub issue for discussion instead of patching.

- Adding `async` to make something cleaner. `async` is viral; the function-coloring problem is exactly what this design rejects.
- Adding a `Result` variant for a recoverable condition. Restructure the types or the boundary instead.
- Adding a separate transaction manager, sync coordinator, GC process, or migration tool. These are absorbed structurally — by compound inputs, by linearization, by treating schema as operations in the log. If a task seems to require one of these, the design is being misread.
- Reaching for `dyn Trait` in hot paths. Monomorphize first. Reserve dynamic dispatch for genuine plugin boundaries.
- Bolting CRDT semantics onto an existing engine. Composition is through trait boundaries, not extension through hooks.
- Defensive runtime checks for what the type system can enforce. If a runtime check is needed at a layer boundary, the boundary is wrong.

## Conventions

- **Branches.** Feature branches off `main`. Squash-merge to `main`.
- **Commits.** Imperative-mood subjects (`add foo`, not `added foo`). Body explains *why* when not obvious from the diff.
- **PRs.** One concern per PR. If a task implies secondary changes, open them as separate PRs.
- **Substantive changes.** Architectural changes — a new layer, a change to an invariant, sync protocol or foldable-trait or storage format decisions, anything touching `docs/design/` — are discussed in a GitHub issue before implementation.
- **Licensing.** Contributions are dual-licensed under MIT and Apache-2.0 by default. See `CONTRIBUTING.md`.
- **Code of conduct.** All participants are bound by `CODE_OF_CONDUCT.md`.

## Build and test

The repository is pre-implementation. Standard Cargo commands apply once code lands:

```text
cargo build
cargo test
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
```

This section will be expanded with project-specific commands as the toolchain stabilizes.

## Where to look

| Concern | Source |
|---|---|
| Architectural worldview, layer stack, invariants | `ARCHITECTURE.md` |
| Per-layer design (in progress) | `docs/design/` |
| Social conventions, how to participate | `CONTRIBUTING.md` |
| Code of conduct | `CODE_OF_CONDUCT.md` |
| Security policy | `SECURITY.md` |

## Attribution discipline

When work touches the architecture, distinguish between what is **designed** (statements grounded in the project's specification or maintainer guidance) and what is **inferred** (logical extrapolation from designed material). Prefer to flag uncertainty over projecting confidence in extrapolations. The architecture is a moving design with stable invariants and contested edges; treat the edges as contested.

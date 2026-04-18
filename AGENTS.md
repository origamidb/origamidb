# AGENTS.md

Guidance for LLM-assisted contributors working on OrigamiDB.

OrigamiDB is an embeddable database with a distributed consensus model drawing from ideas used in CRDTs, built in layers of composed Rust traits. The repository is pre-implementation. There is no source code to modify yet beyond a stub `src/lib.rs`. Most contributions at this stage are documentation, RFCs, and the early trait scaffolding.

For the worldview — the layer stack, invariants, and cross-cutting concerns — read `ARCHITECTURE.md`. This file is short and focused on what an agent needs to know to work productively here without violating the design.

## Hard rules

These are non-negotiable architectural invariants. Patches that violate any of them are wrong by construction.

1. **Mutations are confined to `PageStore`.** Every layer above operates on shared references (`&`). Only `PageStore` holds `&mut`. The borrow checker enforces this; do not work around it with interior mutability or unsafe code.

2. **Trait methods are synchronous.** The shape is `(&mut self, input) -> Option<Output>`. No `async`, no threads, no channels, no blocking inside any layer of the stack.

3. **No `Result<T, E>` for soft errors.** Operations either complete or the process terminates. Conditions that look like errors are either normal (an operation rejected by an invariant predicate in the fold, returning `None`) or fatal (panic and restart from the frontier cache). Do not introduce error variants for partial or recoverable failure.

4. **Operations need not commute.** The convergence layer linearizes concurrent operations during sync. By the time the merge function runs, ordering is already decided. Do not reach for vector clocks, commutativity proofs, or tombstone management inside the merge trait.

5. **The fold constructs state, it does not read.** Reads query the constructed state, not the operation log. Do not implement read paths that scan the log.

6. **The core is `no_std`-friendly.** Reach for `alloc` only behind a feature flag. The lower layers should remain allocation-free where the data structure permits.

## Anti-patterns to flag, not to apply

When a task seems to require any of these, stop and propose an RFC instead of patching.

- Adding `async` to make something cleaner. `async` is viral; the function-coloring problem is exactly what this design rejects.
- Adding a `Result` variant for a recoverable condition. Restructure the types or the boundary instead.
- Adding a separate transaction manager, sync coordinator, GC process, or migration tool. These are absorbed structurally — by compound inputs, by linearization, by the frontier cache, by treating schema as operations in the log. If a task seems to require one of these, the design is being misread.
- Reaching for `dyn Trait` in hot paths. Monomorphize first. Reserve dynamic dispatch for genuine plugin boundaries.
- Bolting CRDT semantics onto an existing engine. Composition is through trait boundaries, not extension through hooks.
- Defensive runtime checks for what the type system can enforce. If a runtime check is needed at a layer boundary, the boundary is wrong.

## Conventions

- **Branches.** Feature branches off `main`. Squash-merge to `main`.
- **Commits.** Imperative-mood subjects (`add foo`, not `added foo`). Body explains *why* when not obvious from the diff.
- **PRs.** One concern per PR. If a task implies secondary changes, open them as separate PRs.
- **Substantive changes.** Architectural changes — a new layer, a change to an invariant, sync protocol or merge algorithm or storage format decisions, anything touching `docs/spec/` — go through the proposal process under `rfcs/`.
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
| Specification (in progress) | `docs/spec/` |
| Proposal process for substantive changes | `rfcs/` |
| Social conventions, how to participate | `CONTRIBUTING.md` |
| Code of conduct | `CODE_OF_CONDUCT.md` |
| Security policy | `SECURITY.md` |

## Attribution discipline

When work touches the architecture, distinguish between what is **designed** (statements grounded in the project's specification or maintainer guidance) and what is **inferred** (logical extrapolation from designed material). Prefer to flag uncertainty over projecting confidence in extrapolations. The architecture is a moving design with stable invariants and contested edges; treat the edges as contested.

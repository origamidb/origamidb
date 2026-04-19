# Contributing to OrigamiDB

Welcome. OrigamiDB is pre-implementation; most contributions today are documentation, RFCs, and the early trait scaffolding that the design depends on. Code contributions open up as the layers land.

## Discussion

- [Discord](https://discord.gg/2JSun56hFY) for informal questions and design discussion.
- [GitHub Issues](https://github.com/origamidb/origamidb/issues) for durable bug reports, feature requests, and tracked work.

## Proposing a change

- **Small fixes, documentation improvements, obvious cleanups** — open a pull request directly. Include enough context in the PR description that a reviewer understands *why*.
- **Architectural changes** — a new layer, a change to an invariant, sync protocol or foldable-trait or storage format decisions, anything touching `docs/design/` — open a GitHub issue for discussion first. Substantive changes require maintainer review before implementation begins.

If you're unsure whether your change needs an RFC, ask in Discord or open a draft issue.

## Pull requests

- Branch off `main`. Squash-merge to `main`.
- Imperative-mood commit subjects (`add foo`, not `added foo`). The body explains *why* when not obvious from the diff.
- One concern per PR. If a task implies secondary changes, propose them as separate PRs.
- Tests when there is something to test. CI when there is something to check.

## License

By contributing to OrigamiDB you agree that your contribution is dual-licensed under MIT and Apache-2.0 — see the [License section of the README](README.md#license). No contributor license agreement is required; the inbound license matches the outbound license.

## Code of Conduct

All participants are bound by the [Code of Conduct](CODE_OF_CONDUCT.md). Report violations to `conduct@asaddu.com`.

## LLM-assisted contributors

If you are driving an AI coding agent — Claude Code, Cursor, Codex, Aider, or similar — point it at [`AGENTS.md`](AGENTS.md). It encodes the architectural invariants and conventions an agent needs to operate here without violating the design.

# origamidb

[![License: MIT or Apache-2.0](https://img.shields.io/badge/license-MIT%20or%20Apache--2.0-blue?style=flat-square)](#license)
![Status: pre-implementation](https://img.shields.io/badge/status-pre--implementation-orange?style=flat-square)
[![Discord](https://img.shields.io/discord/1494838577526341742?label=Discord&logo=discord&logoColor=white&style=flat-square&color=5865F2)](https://discord.gg/2JSun56hFY)
[![GitHub last commit](https://img.shields.io/github/last-commit/origamidb/origamidb?style=flat-square)](https://github.com/origamidb/origamidb/commits/main)

an embeddable database with a distributed consensus model drawing from ideas used in CRDTs.

origamidb is built in layers of composed Rust traits. porting to a new storage medium requires implementing one trait — `PageStore` — at the bottom of the stack. the write path is a deterministic synchronous state machine; side effects are returned as intent for an outer runtime to reconcile, which keeps the core portable across runtime environments and machine architectures.

> **Status:** pre-implementation. the design is captured in [`ARCHITECTURE.md`](ARCHITECTURE.md) and elaborated under [`docs/`](docs/). there is no source code yet beyond a stub.

## Design

[`ARCHITECTURE.md`](ARCHITECTURE.md) is the worldview — the layer stack, the architectural invariants, and the cross-cutting concerns. Deeper specification lives in [`docs/spec/`](docs/spec/) (in progress). Substantial proposals go through the RFC process under [`rfcs/`](rfcs/); decisions are recorded as ADRs under [`docs/adr/`](docs/adr/).

## Community

Discussion happens on [Discord](https://discord.gg/2JSun56hFY). See [`CONTRIBUTING.md`](CONTRIBUTING.md) for how to participate. If you are driving an AI coding agent, point it at [`AGENTS.md`](AGENTS.md).

## License

Licensed under either [Apache-2.0](LICENSE-APACHE) or [MIT](LICENSE-MIT) at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.

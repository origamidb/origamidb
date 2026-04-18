# origamidb

[![License: MIT or Apache-2.0](https://img.shields.io/badge/license-MIT%20or%20Apache--2.0-blue?style=flat-square)](#license)
![Status: pre-implementation](https://img.shields.io/badge/status-pre--implementation-orange?style=flat-square)
[![Discord](https://img.shields.io/discord/1494838577526341742?label=Discord&logo=discord&logoColor=white&style=flat-square&color=5865F2)](https://discord.gg/2JSun56hFY)
[![GitHub last commit](https://img.shields.io/github/last-commit/origamidb/origamidb?style=flat-square)](https://github.com/origamidb/origamidb/commits/main)

origami is an embeddable database

it aims to implement a novel distributed consensus model drawing from ideas used in CRDTs

it's built in layers of composed traits with the intent that porting to a new storage medium simply requires implementing a page store

the write path acts as a deterministic state machine with mutations flowing downwards through the layers. side effects are handled by returned intent, with the expectation that some outer network loop reconciles intent with side effect. as such, the core can be kept extremely portable to different runtime environments and machine architectures

most of the design and architecture goals currently exist in RCS messages and claude context

## Community

Discussion happens on [Discord](https://discord.gg/2JSun56hFY).

## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.

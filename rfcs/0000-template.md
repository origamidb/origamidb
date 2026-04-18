- **Title:** _(short descriptive name, e.g. "Pluggable PageStore for WASM")_
- **Start date:** _(YYYY-MM-DD)_
- **RFC PR:** _(filled in once the pull request is opened)_
- **Tracking issue:** _(filled in if implementation is tracked separately)_

# RFC title

## Summary

One paragraph explanation of what this RFC proposes.

## Motivation

Why is this change needed? What problem does it solve, or what opportunity does it open up, that the current design does not? Include specific use cases where applicable.

## Affected layers

Which layer(s) of the stack does this proposal touch? Check all that apply.

- [ ] PageStore
- [ ] KVStore
- [ ] RelationalStore
- [ ] CRDTStore
- [ ] Partitioning
- [ ] Application boundary
- [ ] Cross-cutting (error model, security, sync protocol, fold semantics, etc.)

## Design

The substance of the proposal.

For trait or interface changes, include the proposed signatures.

For protocol or algorithm changes, describe the mechanics in enough detail that a reviewer can reason about correctness.

For cross-cutting changes, identify the invariants the change preserves, relaxes, or adds.

State the design positively. Save discussion of alternatives for the *Rationale and alternatives* section below.

## Drawbacks

Why might this be a bad idea? Be honest about tradeoffs.

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs were considered, and what is the rationale for not choosing them?
- What is the impact of not doing this?

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this is merged?
- What parts do you expect to resolve through implementation?
- What related issues are out of scope for this RFC and could be addressed independently in the future?

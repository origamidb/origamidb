# OrigamiDB RFCs

This directory holds design documents for substantial changes to OrigamiDB.

## When to open an RFC

- A new trait layer, or a change to an existing layer's responsibilities.
- A change to the universal interface shape or to any architectural invariant stated in [`../ARCHITECTURE.md`](../ARCHITECTURE.md).
- A sync protocol, merge algorithm, or storage format decision.
- Anything that touches `docs/spec/`.

**Not needed for:** bug fixes, small additions to an existing implementation, documentation typos, test improvements. Open a regular pull request for those — see [`../CONTRIBUTING.md`](../CONTRIBUTING.md).

If you are unsure, ask in [Discord](https://discord.gg/2JSun56hFY) or open a draft issue.

## Process

1. **Draft.** Copy [`0000-template.md`](0000-template.md) to `text/0000-my-proposal.md`. Open a pull request. Discussion happens in PR review and as commits to the branch.
2. **Final comment period.** When discussion has converged, a maintainer applies the `final-comment-period` label. The window is seven days. Anyone can object during the window.
3. **Accepted.** Merge the pull request. Rename the file from `0000-` to the PR number (e.g. `text/0042-my-proposal.md`). The RFC is now canonical. Implementation is tracked in a linked issue.

If discussion stalls, the pull request is closed with one of:

- **Postponed** — good idea, wrong time. May be revisited; the conversation is preserved.
- **Withdrawn** — the proposal is no longer under consideration.

In both cases the closing comment notes the status.

## Repository layout

```text
rfcs/
├── README.md           this file
├── 0000-template.md    copy this to start a new RFC
└── text/               accepted RFCs, one per merged PR
```

`text/` is empty until the first RFC merges.

## Amending accepted RFCs

Accepted RFCs are historical records of a decision made at a point in time. They should not be edited substantially after merge. Subsequent changes happen through new RFCs that reference the prior one.

Minor corrections — typos, dead links, formatting — can land as regular pull requests against the accepted text.

## Process references

This process is adapted from [rust-lang/rfcs](https://github.com/rust-lang/rfcs), trimmed for current project scale. The `Postponed` status is borrowed from [Ember RFCs](https://github.com/emberjs/rfcs).

# Architecture Decision Records

This folder holds OrigamiDB's architecture decision records (ADRs). An ADR captures a single architectural decision: the context that motivated it, the alternatives considered, and the consequences.

ADRs are append-only. Once accepted, an ADR is not edited substantively. Decisions that change happen through new ADRs that supersede prior ones.

## When to write an ADR

- A choice between architectural alternatives that the project should remember (why X was picked over Y).
- A constraint or invariant that has been adopted (e.g. "soft errors are not permitted in the engine").
- A change to a previously accepted decision (the new ADR supersedes the old).

If the choice is part of a larger proposal, that proposal probably belongs as an RFC under [`../../rfcs/`](../../rfcs/) instead. ADRs may be derived from accepted RFCs to capture load-bearing decisions in a form that's easy to reference.

## Process

1. **Draft.** Copy [`0000-template.md`](0000-template.md) to `text/0000-short-title.md`. Fill in the sections. Open a pull request.
2. **Discussion.** Discussion happens in PR review. The ADR may be revised as commits to the branch.
3. **Accepted.** Merge the pull request. Rename the file from `0000-` to the PR number (e.g. `text/0042-short-title.md`). Update the status field to `Accepted`. The ADR is now canonical.

If discussion stalls, the pull request is closed without merging. A closed (unmerged) ADR is not part of the record; reopen with a new PR if the decision is revisited.

## Statuses

- **Proposed** — under discussion, not yet decided.
- **Accepted** — adopted; canonical until superseded.
- **Superseded by NNNN** — replaced by a later ADR; left in place for history.
- **Deprecated** — no longer applies, but not directly replaced.

Status changes after acceptance happen by writing a new ADR (which references the prior one) and updating the prior ADR's status field as a small follow-up PR.

## Repository layout

```text
docs/adr/
├── README.md           this file
├── 0000-template.md    copy this to start a new ADR
└── text/               accepted ADRs, one per merged PR
```

`text/` is empty until the first ADR is accepted.

## Format

OrigamiDB ADRs follow a lightweight variant of [MADR](https://adr.github.io/madr/) (Markdown Architecture Decision Records). The template gives the required sections.

## Relationship to RFCs

ADRs and RFCs are complementary. An RFC argues for a substantive change and goes through a final comment period before acceptance — see [`../../rfcs/README.md`](../../rfcs/README.md). An ADR is a decision record: it captures what the project decided and why, in a more compact form.

Some flows:

- Small architectural choice that doesn't warrant a full RFC: write an ADR directly.
- Substantive proposal with multiple architectural choices: write an RFC; on acceptance, derive ADRs for the load-bearing decisions if they will be referenced from elsewhere.
- Decision overturned: write a new ADR that supersedes the prior one. If the change is substantial, it may also warrant an RFC.

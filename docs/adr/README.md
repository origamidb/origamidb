# Architecture Decision Records

This folder holds OrigamiDB's architecture decision records (ADRs). An ADR captures a single architectural decision: the context that motivated it, the alternatives considered, and the consequences.

ADRs are append-only. Once accepted, an ADR is not edited substantively. Decisions that change happen through new ADRs that supersede prior ones.

## When to write an ADR

- A choice between architectural alternatives that the project should remember (why X was picked over Y).
- A constraint or invariant that has been adopted (e.g. "soft errors are not permitted in the engine").
- A change to a previously accepted decision (the new ADR supersedes the old).

## Process

1. **Draft.** Copy [`0000-template.md`](0000-template.md) to `0000-short-title.md`. Fill in the sections. Open a pull request.
2. **Discussion.** Discussion happens in PR review. The ADR may be revised as commits to the branch.
3. **Accepted.** Merge the pull request. Rename the file from `0000-` to the next sequential integer one higher than the highest accepted ADR number (e.g. `0003-short-title.md`). When multiple ADRs land in a single pull request, number them sequentially in the order they are intended to be read. Update the status field to `Accepted`. The ADR is now canonical.

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
└── NNNN-*.md           accepted ADRs, numbered sequentially
```

ADRs live flat in this directory alongside the template, matching MADR convention.

## Format

OrigamiDB ADRs follow a lightweight variant of [MADR](https://adr.github.io/madr/) (Markdown Architecture Decision Records). The template gives the required sections.

Machine-readable YAML frontmatter (the structured-MADR pattern used by some projects to make ADRs indexable by tooling) is under consideration. If adopted, the template and this README will be updated to reflect the chosen schema.

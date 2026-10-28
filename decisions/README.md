# Decision Log (ADR)

This folder records Legoin decisions in [MADR](https://adr.github.io/madr/) format, one decision per file. It covers every kind of decision: product, design, technical and process.

## When to write an ADR

- The decision is hard to reverse or affects more than one person.
- There are several reasonable options, and the reason for the choice is worth remembering.
- The decision is already in the code or design but not recorded. Write it retroactively and say so.

## How to write one

1. Copy [template.md](template.md) to `NNNN-short-title.md`, where `NNNN` is the next 4-digit number and the title is kebab-case.
2. Fill in every section. Start with status `Proposed`.
3. Open a pull request. The discussion happens there.
4. Before merging, set the status to `Accepted` or `Rejected`.
5. Add a row to the [index](#index).

## Status

| Status | Meaning |
| --- | --- |
| `Proposed` | Suggested, not yet decided |
| `Accepted` | Approved and in effect |
| `Rejected` | Turned down; kept for the record |
| `Deprecated` | No longer in effect, with no replacement |
| `Superseded` | Replaced by another ADR (link it) |

Don't change the content of an `Accepted` ADR. To change a decision, write a new ADR that supersedes the old one and update the old one's status. Fixing typos and links is fine.

## Index

| No | Title | Status | Date |
| --- | --- | --- | --- |
| [0001](0001-web-tech-stack.md) | Web frontend tech stack | Accepted | 2026-10-28 |
| [0002](0002-visual-direction.md) | Instruction-booklet style with LEGO red and yellow | Superseded by 0003 | 2026-10-28 |
| [0003](0003-ballers-style-lego-colors.md) | Ballers-addict style in LEGO colors, Dela Gothic One and the name Legoin | Accepted | 2026-10-28 |
| [0004](0004-english-default-with-indonesian.md) | English by default, Indonesian as a second language | Accepted | 2026-10-28 |
| [0005](0005-ml-repository-and-model-releases.md) | Models in legoin-ml, weights as GitHub Releases | Proposed | 2026-10-28 |

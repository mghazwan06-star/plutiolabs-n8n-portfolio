# Engineering

How these systems were built, audited, and repaired.

| Document | What it covers |
|---|---|
| [**Bug taxonomy**](bug-taxonomy.md) | 111 defects in 9 recurring classes, with root causes and the reasoning that missed them |
| [**Audit methodology**](audit-methodology.md) | 17 audit passes, ranked by what each class actually caught |
| [**Build standards**](build-standards.md) | The conventions enforced across all 41 workflows, each traced to its originating defect |
| [**How Claude was used**](how-claude-was-used.md) | Claude Code + n8n MCP as the build environment — including where it failed |

## The one-line version

> 41 workflows. 673 nodes. 17 audit passes. 111 logged defects.
> **Static validation caught none of them.**

Everything in this directory follows from that. Structural validity and behavioural correctness are
different properties, and only one of them is machine-checkable today. The defects that reached
production were timezone arithmetic, silent column-name mismatches, unreachable error branches,
concurrency races, and unauthenticated endpoints — all structurally valid, all wrong.

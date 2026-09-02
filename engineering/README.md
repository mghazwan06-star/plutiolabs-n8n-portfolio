# Engineering

How these systems were built, audited and repaired.

| Document | What it covers |
|---|---|
| [**Bug taxonomy**](bug-taxonomy.md) | 111 bugs in nine recurring classes, with root causes and the reasoning that missed them |
| [**Audit methodology**](audit-methodology.md) | 17 audit passes ranked by what each kind actually caught |
| [**Build standards**](build-standards.md) | The conventions used across all 41 workflows, each traced to the bug that caused it |
| [**How Claude was used**](how-claude-was-used.md) | Claude Code and an n8n MCP server as the build environment, including where it failed |

## Short version

41 workflows. 673 nodes. 17 audit passes. 111 logged bugs. Static validation caught none of them.

Everything here follows from that. Structural validity and behavioural correctness are different
properties and only one of them is machine-checkable today. The bugs that reached production were
timezone arithmetic, silent column-name mismatches, unreachable error branches, concurrency races
and unauthenticated endpoints. All structurally valid. All wrong.

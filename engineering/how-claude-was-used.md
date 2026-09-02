# How Claude Was Used

I built every workflow here inside Claude Code, driving my live n8n instance through an MCP server.
Not "generated with AI assistance": the model held the API connection and did the building.

This page includes where that failed.

## Setup

```
Claude Code  --MCP-->  n8n-mcp server  --REST-->  live n8n instance
     |
     +-- 20 project skills  (n8n patterns, node config, expressions, validation,
     |                       VAPI integration, GDPR, prompt engineering)
     +-- CLAUDE.md          (build protocol, technical rules, safety constraints)
```

**MCP server.** [`n8n-mcp`](https://github.com/czlonkowski/n8n-mcp) exposes about 19 tools over a
live instance: node documentation and search across 1,000+ node types, config validation, template
search, and full workflow CRUD including create, partial update, validate, execute and execution
history.

Discovery is the half that matters. Configuring an n8n node from memory produces plausible, wrong
JSON: property names that don't exist, operations under the wrong resource, missing required fields.
So my rule became `search_nodes`, then `get_node(mode: "docs")`, then `validate_node`, before adding
any node. Never configure from memory.

**Skills.** I wrote 20 skills as reference material, loaded per build phase rather than held in context: n8n
architectural patterns, node configuration, expression syntax, Code node JavaScript, validation
triage, VAPI payload formats, UK GDPR and PECR requirements.

**CLAUDE.md.** My project rules: a six-phase build protocol, the technical standards from
[`build-standards.md`](build-standards.md), and hard safety constraints. Never modify a production
workflow without confirming, always read before writing, prefer partial updates.

## The build protocol

| Phase | Action |
|---|---|
| 0 Context | Read the build rules, the CRM schema, the plan. Map data contracts before touching a tool. |
| 1 Pattern | Pick an architecture. Map every node before building any of them. |
| 2 Research | Per node type: `search_nodes`, `get_node`, `validate_node`. |
| 3 Build | Trigger, sticky note, config node, then nodes in execution order. Partial updates only. |
| 4 Validate | Strict profile. Fix every real error, triage every warning against known false positives. |
| 5 Cross-check | Verify column names against the schema, verify contracts between workflows. |
| 6 Test | Execute. Inspect output. Confirm the response format. |

I added phase 5 because phases 0 to 4 kept producing workflows that were internally valid and wrong
at the boundaries.

## What worked

**Node discovery.** My biggest single accuracy gain. Looking up a node's real schema before
configuring it removed an entire class of hallucinated-property errors.

**Consistency sweeps at scale.** "Find every Sheets node across 40 workflows writing a column name
that isn't in the schema" is tedious, mechanical work that an agent with API access is genuinely
good at. It was my highest-yield audit technique.

**Documentation written during the build.** An 82KB fix log, three audit protocols, per-workflow
testing evidence, all written as I went rather than reconstructed afterwards. This repository exists
because I kept that record.

**Adversarial self-review, when I asked for it explicitly.** Told to attack its own output ("find the
phantom-success branches", "find every unreachable IF"), the model found real bugs in workflows it
had written itself and previously called clean.

## What did not work

This is the part of the page worth reading.

**Static validation created false confidence.** The validator returned 0 errors on workflows whose
booking system was entirely dead. I kept reading a clean report as "correct" when it means "well
formed". 111 bugs, none of them caught by it.

**I was told "swept clean" twice before the worst bug turned up.** The escaped-literal bug that
killed booking product-wide surfaced in a leftovers review after two confident all-clear reports.
Both came off a clean validator run and a completed checklist. Neither was true.

**Fixing three of four identical nodes.** This happened five separate times. Pattern-matching across
many similar items degrades in exactly the way a checklist prevents, which is why I replaced "fix the
pattern" with enumerate-and-tick.

**A made-up constant lasted two months.** My build rules asserted a 5-second VAPI tool timeout and
neither of us had checked the vendor docs. The real value is 20 seconds by default. That invented
number drove real architecture decisions, including deferring a fix that didn't need deferring. It
survived because it was written down confidently and never sourced.

**Reasoning from stale local copies.** I got two bug reports that described nothing real, derived
from an out-of-date local file instead of the deployed node.

**API success is not stored state.** An `updateNode` call returning `success` while storing a
double-escaped string is the most consequential lesson here. Verification means reading the value
back.

**And once more, publishing this.** The sanitizer I wrote to strip secrets from these exports passed
its own verification twice, then GitHub's secret scanning blocked my first push over a Twilio Account
SID I had no pattern for. Same lesson as the rest of this page, found the same way: by something
outside my own loop checking the work.

## Honest summary

Claude Code with MCP access let me build, audit and document 40 production workflows across four
connected systems in eight months on my own, including the cross-workflow consistency work I would
not realistically have done by hand at this scale.

It did not make the work correct. That came from adversarial review, mechanical sweeps, concurrency
testing, and reading things back after writing them. The model was very good at executing those
methods once I specified them, and consistently overconfident about whether it had finished.

Both halves are in this repository: 40 working workflows, and 111 logged reasons the first version of
each was wrong.

## Takeaways

1. Give the agent live API access to the target system. The difference between generating config and
   validating it against a real schema is most of the accuracy.
2. Make discovery mandatory. "Never configure from memory" belongs in the project rules.
3. Write the failure log during the build. It is worth more than the code afterwards.
4. Ask for adversarial review explicitly. "Review this" gets me agreement. "Find the branches that
   report success without checking" gets me findings.
5. Never treat a clean validator as proof of correctness. It proves well-formedness.
6. Read back what you wrote, especially through an API and especially with non-ASCII identifiers.

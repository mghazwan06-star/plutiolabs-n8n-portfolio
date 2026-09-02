# How Claude Was Used

Every workflow here was built inside Claude Code, driving a live n8n instance through an MCP server.
Not "generated with AI assistance". The model held the API connection and did the building.

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
The rule became `search_nodes`, then `get_node(mode: "docs")`, then `validate_node`, before adding
any node. Never configure from memory.

**Skills.** 20 skills as reference material loaded per build phase rather than held in context: n8n
architectural patterns, node configuration, expression syntax, Code node JavaScript, validation
triage, VAPI payload formats, UK GDPR and PECR requirements.

**CLAUDE.md.** The project's rules: a six-phase build protocol, the technical standards from
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

Phase 5 exists because phases 0 to 4 kept producing workflows that were internally valid and wrong
at the boundaries.

## What worked

**Node discovery.** The biggest single accuracy gain. Looking up a node's real schema before
configuring it removed an entire class of hallucinated-property errors.

**Consistency sweeps at scale.** "Find every Sheets node across 41 workflows writing a column name
that isn't in the schema" is tedious, mechanical work that an agent with API access is genuinely
good at. It was the highest-yield audit technique in the project.

**Documentation written during the build.** An 82KB fix log, three audit protocols, per-workflow
testing evidence, all written as the work happened rather than reconstructed afterwards. This
repository exists because that record existed.

**Adversarial self-review, when asked for explicitly.** Told to attack its own output ("find the
phantom-success branches", "find every unreachable IF"), the model found real bugs in workflows it
had written itself and previously called clean.

## What did not work

This is the part of the page worth reading.

**Static validation created false confidence.** The validator returned 0 errors on workflows whose
booking system was entirely dead. A clean report kept getting read as "correct" when it means "well
formed". 111 bugs, none of them caught by it.

**"Swept clean" was reported twice before the worst bug was found.** The escaped-literal bug that
killed booking product-wide turned up in a leftovers review after two confident all-clear reports.
Both were made in good faith off a clean validator run and a completed checklist. Neither was true.

**Fixing three of four identical nodes.** Happened five separate times. Pattern-matching across many
similar items degrades in exactly the way a checklist prevents, which is why enumerate-and-tick
replaced "fix the pattern".

**A made-up constant lasted two months.** The build rules asserted a 5-second VAPI tool timeout.
Nobody had checked the vendor docs. The real value is 20 seconds by default. That invented number
drove real architecture decisions, including deferring a fix that didn't need deferring. It survived
because it was written down confidently and never sourced.

**Reasoning from stale local copies.** Two bugs were reported that did not exist, derived from an
out-of-date local file instead of the deployed node.

**API success is not stored state.** An `updateNode` call returning `success` while storing a
double-escaped string is the most consequential lesson here. Verification means reading the value
back.

**And once more, while building this repository.** The sanitizer I wrote to strip secrets from these
exports passed its own verification twice, then GitHub's secret scanning blocked the first push over
a Twilio Account SID I had no pattern for. Same lesson as the rest of this page, found the same way:
by something outside the loop checking the work.

## Honest summary

Claude Code with MCP access made it possible for one person to build, audit and document 41
production workflows across five connected systems in eight months, including the cross-workflow
consistency work that realistically doesn't get done by hand at this scale.

It did not make the work correct. Correctness came from adversarial review, mechanical sweeps,
concurrency testing, and reading things back after writing them. The model was very good at
executing those methods once they were specified, and consistently overconfident about whether it
had finished.

Both halves are in this repository: 41 working workflows, and 111 logged reasons the first version
of each was wrong.

## Takeaways

1. Give the agent live API access to the target system. The difference between generating config and
   validating it against a real schema is most of the accuracy.
2. Make discovery mandatory. "Never configure from memory" belongs in the project rules.
3. Write the failure log during the build. It is worth more than the code afterwards.
4. Ask for adversarial review explicitly. "Review this" produces agreement. "Find the branches that
   report success without checking" produces findings.
5. Never treat a clean validator as proof of correctness. It proves well-formedness.
6. Read back what you wrote, especially through an API and especially with non-ASCII identifiers.

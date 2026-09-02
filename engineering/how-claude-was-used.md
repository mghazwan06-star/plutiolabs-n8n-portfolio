# How Claude Was Used

Every workflow in this repository was built inside **Claude Code**, driving a live n8n instance
through an **MCP server**. Not "generated with AI assistance" — the model held the API connection
and did the building.

This page describes the setup honestly, including where it failed.

---

## The setup

```
Claude Code  ──MCP──▶  n8n-mcp server  ──REST──▶  live n8n instance
     │
     ├── 20 project skills  (n8n patterns, node config, expressions, validation,
     │                       VAPI integration, GDPR, prompt engineering, …)
     └── CLAUDE.md          (build protocol, technical rules, safety constraints)
```

**MCP server.** [`n8n-mcp`](https://github.com/czlonkowski/n8n-mcp) exposes ~19 tools over a live
n8n instance: node documentation and search across 1,000+ node types, config validation, template
search, and full workflow CRUD — create, partial-update, validate, execute, inspect execution
history.

The important half is *discovery*. Configuring an n8n node from memory produces plausible,
wrong JSON: property names that don't exist, operations under the wrong resource, required fields
omitted. The standing rule became `search_nodes` → `get_node(mode: "docs")` → `validate_node`
**before** adding any node. Never configure from memory.

**Skills.** 20 skills as progressively-loaded reference material — n8n architectural patterns,
node configuration, expression syntax, Code node JavaScript, validation triage, VAPI payload
formats, UK GDPR/PECR requirements. Loaded per build phase rather than held in context.

**CLAUDE.md.** The project's constitution: a six-phase build protocol, the technical rules from
[`build-standards.md`](build-standards.md), and hard safety constraints — never modify a production
workflow without confirmation, always read before writing, prefer partial updates.

---

## The build protocol

Six phases, applied to every workflow:

| Phase | Action |
|---|---|
| 0 · Context | Read the build rules, the CRM schema, the plan. Map upstream/downstream data contracts *before* touching a tool. |
| 1 · Pattern | Select an architectural pattern. Map every node before building any of them. |
| 2 · Research | Per node type: `search_nodes` → `get_node` → `validate_node`. |
| 3 · Build | Trigger, sticky note, config node, then nodes in execution order. Partial updates only. |
| 4 · Validate | `strict` profile. Fix every real error; triage every warning against a known-false-positive list. |
| 5 · Cross-check | Verify every column name against the canonical schema; verify data contracts between workflows. |
| 6 · Test | Execute. Inspect output. Confirm response format. |

Phase 5 exists because phases 0–4 kept producing workflows that were internally valid and wrong at
the boundaries.

---

## What worked well

**Node discovery.** The largest single accuracy gain. Looking up a node's real schema before
configuring it eliminated an entire class of hallucinated-property errors.

**Consistency sweeps at scale.** "Find every Sheets node across 41 workflows that writes a column
name not in the schema" is tedious, mechanical, and exactly what an agent with API access is good
at. This was the highest-yield audit technique in the project.

**Documentation as a build artifact.** 82KB fix log, three audit protocols, per-workflow testing
evidence — written *during* the build, not reconstructed after. This repository exists because
that record existed.

**Adversarial self-review.** Given an explicit instruction to attack its own output — "find the
phantom-success branches," "find every unreachable IF" — the model found real defects in workflows
it had written itself and previously declared clean.

---

## What did not work

This section is the point of the page.

**Static validation created false confidence.** The validator returned 0 errors on workflows whose
booking system was entirely dead. A clean report was repeatedly read as "correct," and it means
"well-formed." 111 defects, 0 caught by it.

**"Swept clean" was reported twice before the worst defect was found.** The escaped-literal defect
that killed booking product-wide (Class 8) was found in a leftovers review *after* two confident
all-clear reports. Both reports were made in good faith, based on a clean validator run and a
completed checklist. Neither was true.

**The 3-of-4 sibling miss.** Fixing three of four identical nodes and missing one happened five
separate times. Pattern-matching across many similar items degrades in exactly the way a checklist
prevents — which is why enumerate-and-tick replaced "fix the pattern."

**A fabricated constant lived for two months.** The build rules asserted a 5-second VAPI tool
timeout. Nobody had checked the vendor documentation; the real value is 20 seconds by default. That
invented number drove real architecture decisions, including deferring a fix that didn't need
deferring. It survived because it was written down confidently and never sourced.

**Reasoning from stale local copies.** Two defects were reported that did not exist — derived from
an out-of-date local file instead of the deployed node. Hence the rule: read the deployed artefact,
never a local copy.

**API success ≠ stored state.** An `updateNode` call returning `success` while storing a
double-escaped string is the most consequential lesson here. Verification means reading the value
back.

---

## The honest summary

Claude Code with MCP access made it possible for one person to build, audit and document 41
production workflows across five interconnected systems in eight months — including the tedious
cross-workflow consistency work that is realistically not done by hand at this scale.

It did not make the work correct. Correctness came from adversarial review, mechanical sweeps,
concurrency testing, and reading things back after writing them. The model was excellent at
executing those methods once they were specified and consistently over-confident about whether it
had finished.

Both halves of that are in this repository: 41 working workflows, and 111 logged reasons the first
version of each was wrong.

---

## Reusable takeaways

1. **Give the agent live API access to the target system.** The difference between generating
   config and validating it against a real schema is most of the accuracy.
2. **Make discovery mandatory.** "Never configure from memory" belongs in the project rules.
3. **Write the failure log during the build.** It is worth more than the code afterwards.
4. **Ask for adversarial review explicitly.** "Review this" produces agreement; "find the branches
   that report success without checking" produces findings.
5. **Never accept a clean validator as proof of correctness.** It proves well-formedness.
6. **Read back what you wrote.** Especially through an API, especially with non-ASCII identifiers.

# Audit Methodology

17 audit passes ran across the three largest systems between April and August 2026. This documents
what each *class* of pass was, and — more usefully — what each class actually caught.

## The passes

| System | Passes | Span |
|---|---:|---|
| AI Voice Agent | 5 | Apr 15 → Jun 12 |
| WhatsApp Chatbot | 7 | May 14 → Jun 12 |
| Lead Reactivation | 5 | May 22 → Jun 12 |

Each pass is logged with a date, a scope, the findings, and the fix applied. The logs are
append-only: superseded conclusions stay in the record with a correction beside them, because
*which* wrong assumption was held for how long turned out to be the most reusable information in
the project.

---

## Pass classes, ranked by yield

### 1. Cross-workflow consistency sweeps — highest yield

Pick one confirmed defect. Search every sibling workflow for the same pattern before moving on.

This single practice caught more real defects than any other. A representative case: after fixing
`Cancelled At` → `CancelledAt` in one workflow, the sweep found the same defect in two more —
every cancellation write in the system had been failing silently.

It also exposed a distinct failure mode worth naming: the **3-of-4 sibling miss**. Four nodes share
a pattern, three get fixed, one is missed. This happened five separate times. It is now assumed
rather than hoped against: after any fix, enumerate *all* siblings and check them off individually.

### 2. Adversarial phase testing — highest-value findings

Nine scripted conversation phases run against the live chatbot with evidence captured for each:
opening hours gates, general Q&A, vague and contradictory input, booking, reschedule, cancel,
message-taking, identity/prompt-injection attempts, and opt-out.

This is where every concurrency defect surfaced. Sequential testing passes cleanly; the defects
require *simultaneous* load. The session race in particular was only found by sending messages one
second apart — which required deliberately simulating how a human actually opens a conversation,
not how a test script does.

### 3. Node-property sweeps — high yield, fully mechanical

Enumerate every node of a type and check a required property is present. `onError` on every Code
node, `alwaysOutputData` on every Sheets read, `retryOnFail` on Sheets and Calendar operations,
`fallbackOutput` on every Switch, `includeOtherFields` on every config Set node, `range: "A:Z"` on
every Sheets read.

The validator checks none of these. They are scriptable, and they caught two catastrophic defects:
eight Switch nodes whose rules had been silently wiped by an earlier bulk update, and eight config
Set nodes dropping their incoming payload.

### 4. Schema conformance — targeted and reliable

Diff every column name written by every Sheets node against the canonical schema document. Pure
string comparison, no judgement required, and it catches an entire defect class (Class 1) that is
otherwise invisible until someone notices missing data weeks later.

### 5. Security review — one pass, disproportionate impact

Run once, late. Enumerate every webhook and ask what an unauthenticated caller could do. Found
five open endpoints, including one that places real phone calls, plus an API key stored in
plaintext across six config nodes.

The lesson is scheduling, not technique: this pass ran *after* the systems were considered
finished. It should have run at the first webhook.

### 6. External-agent audits — moderate yield

A second agent, given the workflow JSON and no build history, reviews it cold. Catches things the
author has stopped seeing. Lower hit rate than the sweeps, but the findings are qualitatively
different — assumptions rather than omissions.

### 7. Static validation — zero yield

`n8n_validate_workflow` in `strict` profile, run on every workflow, repeatedly, across all 17
passes. **It never found a defect that mattered.**

This is not a criticism of the tool. It does what it claims: confirms nodes exist, required
parameters are set, connections resolve, expressions parse. It is a necessary gate and a
prerequisite for deployment. It is simply not evidence of correctness, and treating a clean
validator report as such is what let several of these defects live for months.

The distinction matters enough to state plainly: **the manual passes were the foundation; static
validation was the zero-yield activity.** They are easy to conflate because both are called
"auditing."

---

## Recurring procedural rules

These emerged from the passes and are now enforced on every build:

**Read the deployed artefact, never a local copy.** Two defects were reported that did not exist —
invented by reasoning from a stale local scratchpad instead of fetching the live node. Always fetch
current state before claiming a conflict.

**Verify writes by reading them back.** An API `success` response describes the request, not the
stored value. This rule exists because of the escaped-literal defect (Class 8), where five nodes
were confirmed "updated successfully" while storing a string that threw on every execution.

**Never port a rule across systems without a context check.** Rules that are correct in one system
can be actively wrong in another with different failure semantics. `retryOnFail` is the standing
example: correct on Sheets writes, dangerous on a call-placing endpoint. Every ported rule is
verified against the target's architecture and documented where it is deliberately exempt.

**Suspicion of dismissed warnings.** The single worst defect surfaced inside a warning that had
been dismissed as a false positive for three days. False-positive triage is necessary, but a
dismissed warning should be re-read whenever a defect appears near it.

---

## What I would change

**Run the security pass first, not last.** It found the highest-severity issues in the project and
found them late.

**Test concurrency from the start.** Every concurrency defect was found in one late testing phase.
None would have survived a simultaneous-load test written on day one.

**Treat "the docs say X" as a claim requiring a source.** An invented timeout figure sat in the
build rules for two months and drove real architecture decisions before anyone checked it against
the vendor's documentation.

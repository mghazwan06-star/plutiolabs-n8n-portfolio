# Audit Methodology

17 audit passes ran across the three largest systems between April and August 2026. This is what
each kind of pass was, and what each one actually caught.

| System | Passes | Span |
|---|---:|---|
| AI Voice Agent | 5 | 15 Apr to 12 Jun |
| WhatsApp Chatbot | 7 | 14 May to 12 Jun |
| Lead Reactivation | 5 | 22 May to 12 Jun |

Every pass is logged with a date, a scope, the findings and the fix. The logs are append-only.
Superseded conclusions stay in the record with a correction next to them, because which wrong
assumption was held, and for how long, turned out to be the most reusable information in the project.

## Ranked by what they found

### 1. Cross-workflow consistency sweeps

Take one confirmed bug. Search every other workflow for the same pattern before moving on.

This found more real bugs than anything else. After fixing `Cancelled At` to `CancelledAt` in one
workflow, the sweep found it in two more, meaning every cancellation write in the system had been
failing silently.

It also exposed a failure mode worth naming: fixing three of four identical nodes and missing the
fourth. That happened five separate times. It is now assumed rather than hoped against. After any
fix, list all the siblings and tick them off individually.

### 2. Adversarial phase testing

Nine scripted conversation phases run against the live chatbot with evidence saved for each: opening
hours, general Q&A, vague and contradictory input, booking, reschedule, cancel, message taking,
prompt injection attempts, and opt-out.

Every concurrency bug surfaced here. Sequential testing passes cleanly. These needed simultaneous
load, and the session race needed something more specific: simulating how a person actually opens a
conversation rather than how a test script does.

### 3. Node property sweeps

List every node of a type and check a required property is set. `onError` on every Code node,
`alwaysOutputData` on every Sheets read, `retryOnFail` on Sheets and Calendar, `fallbackOutput` on
every Switch, `includeOtherFields` on every config Set, `range: "A:Z"` on every Sheets read.

The validator checks none of these. They are scriptable, and they caught two serious problems: eight
Switch nodes whose rules had been silently wiped by an earlier bulk update, and eight config Set
nodes dropping their incoming payload.

### 4. Schema conformance

Diff every column name written by every Sheets node against the schema document. Pure string
comparison, no judgement needed, and it catches a whole class of bug that is otherwise invisible
until someone notices missing data weeks later.

### 5. Security review

Run once, late. List every webhook and ask what an unauthenticated caller could do. Found five open
endpoints, one of which places real phone calls, plus an API key sitting in plaintext across six
config nodes.

The lesson is about timing rather than technique. This pass ran after the systems were considered
finished. It should have run at the first webhook.

### 6. External agent audits

A second agent reviews the workflow JSON cold, with no build history, and catches things the author
has stopped seeing. Lower hit rate than the sweeps, but the findings are a different kind:
assumptions rather than omissions.

### 7. Static validation

`n8n_validate_workflow` in `strict` profile, run on every workflow, repeatedly, across all 17
passes. It never found a bug that mattered.

That is not a criticism of the tool. It does what it claims: confirms nodes exist, required
parameters are set, connections resolve, expressions parse. It is a necessary gate before
deployment. It is just not evidence of correctness, and treating a clean report as if it were is
what let several of these bugs live for months.

Worth being blunt about, because both activities get called "auditing": the manual passes were the
foundation. Static validation was the part with zero yield.

## Rules that came out of it

**Read the deployed artefact, never a local copy.** Two bugs were reported that did not exist,
invented by reasoning from a stale local scratchpad instead of fetching the live node. Fetch current
state before claiming a conflict.

**Verify writes by reading them back.** An API `success` response describes the request, not the
stored value. This rule exists because of the escaped-literal bug, where five nodes were confirmed
updated while storing a string that threw on every run.

**Never port a rule between systems without checking it fits.** A rule that is right in one system
can be actively wrong in another with different failure semantics. `retryOnFail` is the standing
example: correct on Sheets writes, dangerous on an endpoint that places calls. Every ported rule
gets checked against the target's architecture, and exemptions get documented.

**Re-read dismissed warnings.** The worst bug in the project surfaced inside a warning that had been
dismissed as a false positive for three days. Triage is necessary, but a dismissed warning deserves
a second look whenever a bug turns up near it.

## What I would do differently

**Run the security pass first.** It found the highest-severity problems and found them last.

**Test concurrency from day one.** Every concurrency bug came out of one late testing phase. None
would have survived a simultaneous-load test written at the start.

**Treat "the docs say X" as a claim that needs a source.** An invented timeout figure sat in the
build rules for two months and shaped real decisions before anyone checked it.

# Audit Methodology

I ran 17 audit passes across my three largest systems between April and August 2026. This is what
each kind of pass was, and what each one actually caught.

| System | Passes | Span |
|---|---:|---|
| AI Voice Agent | 5 | 15 Apr to 12 Jun |
| WhatsApp Chatbot | 7 | 14 May to 12 Jun |
| Lead Reactivation | 5 | 22 May to 12 Jun |

I log every pass with a date, a scope, the findings and the fix, and I keep the logs append-only.
Superseded conclusions stay in the record with a correction next to them, because which wrong
assumption I held, and for how long, turned out to be the most reusable thing I wrote down.

## Ranked by what they found

### 1. Cross-workflow consistency sweeps

Take one confirmed bug. Search every other workflow for the same pattern before moving on.

This found more real bugs for me than anything else. After fixing `Cancelled At` to `CancelledAt` in
one workflow, the sweep found it in two more, which meant every cancellation write in the system had
been failing silently.

It also exposed a habit of mine worth naming: fixing three of four identical nodes and missing the
fourth. I did that five separate times. I now assume it rather than hope against it, so after any fix
I list all the siblings and tick them off one by one.

### 2. Adversarial phase testing

Nine scripted conversation phases I ran against the live chatbot, saving evidence for each: opening
hours, general Q&A, vague and contradictory input, booking, reschedule, cancel, message taking,
prompt injection attempts, and opt-out.

Every concurrency bug I have surfaced here. Sequential testing passes cleanly. These needed
simultaneous load, and the session race needed something more specific: simulating how a person
actually opens a conversation rather than how a test script does.

### 3. Node property sweeps

List every node of a type and check a required property is set. `onError` on every Code node,
`alwaysOutputData` on every Sheets read, `retryOnFail` on Sheets and Calendar, `fallbackOutput` on
every Switch, `includeOtherFields` on every config Set, `range: "A:Z"` on every Sheets read.

The validator checks none of these. They are scriptable, and they caught two serious problems for me:
eight Switch nodes whose rules an earlier bulk update had silently wiped, and eight config Set nodes
dropping their incoming payload.

### 4. Schema conformance

I diff every column name written by every Sheets node against my schema document. Pure string
comparison, no judgement needed, and it catches a whole class of bug that is otherwise invisible
until someone notices missing data weeks later.

### 5. Security review

I ran this once, late. List every webhook and ask what an unauthenticated caller could do. It found
five open endpoints, one of which places real phone calls, plus an API key sitting in plaintext
across six config nodes.

The lesson is about timing rather than technique. I ran this pass after I considered the systems
finished. It should have run when I built the first webhook.

### 6. External agent audits

A second agent reviews the workflow JSON cold, with no build history, and catches things the author
has stopped seeing. Lower hit rate than the sweeps, but the findings are a different kind:
assumptions rather than omissions.

### 7. Static validation

`n8n_validate_workflow` in `strict` profile, run on every workflow, repeatedly, across all 17 passes.
It never found a bug that mattered.

That is not a criticism of the tool. It does what it claims: confirms nodes exist, required
parameters are set, connections resolve, expressions parse. It is a necessary gate before
deployment. It is just not evidence of correctness, and treating a clean report as if it
were is what let several of these bugs live for months in my systems.

Worth being blunt, because both get called "auditing": the manual passes were the foundation. Static
validation was the part with zero yield.

## Rules that came out of it

**Read the deployed artefact, never a local copy.** I reported two bugs that did not exist, invented
by reasoning from a stale local scratchpad instead of fetching the live node. I fetch current state
before claiming a conflict now.

**Verify writes by reading them back.** An API `success` response describes the request, not the
stored value. This rule exists because of the escaped-literal bug, where I had five nodes
confirmed as updated while they stored a string that threw on every run.

**Never port a rule between my systems without checking it fits.** A rule that is right in one system
can be actively wrong in another with different failure semantics. `retryOnFail` is the standing
example: correct on Sheets writes, dangerous on an endpoint that places calls. I check every ported rule against the target's
architecture and document the exemptions.

**Re-read dismissed warnings.** My worst bug surfaced inside a warning I had dismissed as a false
positive for three days. Triage is necessary, but a dismissed warning deserves
a second look whenever a bug turns up near it.

## What I would do differently

**Run the security pass first.** It found my highest-severity problems and I ran it last.

**Test concurrency from day one.** Every concurrency bug came out of one late testing phase. None of
them would have survived a simultaneous-load test written at the start.

**Treat "the docs say X" as a claim that needs a source.** A timeout figure I invented sat in my
build rules for two months and shaped real decisions before I checked it.

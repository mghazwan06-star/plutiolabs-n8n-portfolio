# Bug Taxonomy

111 unique defect IDs were logged across the three largest systems between April and August 2026.
This page groups them into recurring classes.

The headline finding is stated once and then evidenced throughout:

> **Static validation caught 0 of them.**

n8n's workflow validator ran in `strict` profile on every workflow, repeatedly, and consistently
reported 0 errors. It validates *structure*: node types exist, required parameters are present,
connections point at real nodes, expressions parse. Every defect below is *semantic* — structurally
valid, and wrong.

---

## Class 1 — Silent write failures

The most dangerous class, because the workflow reports success.

**`Cancelled At` vs `CancelledAt`.** Three workflows wrote a cancellation timestamp to a column
named `Cancelled At` (with a space). The live sheet's column is `CancelledAt`. Google Sheets does
not error on an unmatched column name in an `appendOrUpdate` — it simply doesn't write that field.
Every cancellation across the voice agent recorded successfully and lost its timestamp. Found by
diffing every column name in every Sheets node against the canonical schema file, not by testing.

**Luxon objects where strings were expected.** `{{ $now }}` in a Sheets value resolves to a Luxon
`DateTime` *object*, not a string. Serialised into a cell it produces unusable output. Found in
5 separate nodes — and notably, in 4 of the 5 cases three sibling nodes had been fixed and one
missed, a pattern frequent enough that it got its own audit rule.

**`cellFormat: RAW`.** 36 Sheets writes stored values raw, meaning phone numbers could be coerced
into numeric form and lose their leading `+`. Compliance-adjacent: the opt-out check downstream is
an exact phone-string match, so a reformatted number silently stops matching its own opt-out record.

**Takeaway:** an external system accepting your write is not confirmation it stored what you meant.

---

## Class 2 — Timezone and date arithmetic

**`setHours()` on a UTC server.** Booking logic used `setHours()` to construct appointment times.
n8n Cloud runs UTC; the business runs Europe/London. From late March to late October the UK is on
BST (UTC+1), so **every booking made during British Summer Time was written one hour late** — for
the entire summer. The fix was an explicit BST offset (`month 3–10 → +1, else +0`) applied
consistently across all systems.

**V8 defaults yearless date strings to 2001.** A parser accepted "the 14th of March" and produced
`2001-03-14`, which is in the past, so availability checks returned nothing bookable. This is
JavaScript `Date` behaviour, not an n8n quirk, and it is invisible until you inspect the parsed value.

**Explicit day + month discarded.** A related parser preferred a relative interpretation and threw
away the explicit date the customer had actually given.

**Takeaway:** timezone bugs are seasonal. A build tested in February is not tested for BST.

---

## Class 3 — Phantom success

Branches that report success without checking whether the operation succeeded.

Five sites were found where a response builder returned a success message unconditionally. Two were
serious: the **do-not-contact** confirmation was spoken aloud to a caller by the voice agent before
the opt-out had been verified as written. Under UK GDPR/PECR, telling someone they have been opted
out when they have not is a compliance failure, not a cosmetic one.

Also in this class: **failed reminders and follow-ups marked as sent** and therefore never retried,
and a calendar-read failure that reported the entire day as free — offering slots that were
already booked.

**Takeaway:** a `try/catch` that swallows the error and returns the success path is worse than no
handler, because it destroys the evidence.

---

## Class 4 — Unreachable error handling

Three IF nodes existed specifically to catch calendar failures. Under n8n's **strict
`typeValidation`**, the comparison they used could never evaluate true, so the error branch was
dead code. The workflow looked defensively written and had no defence at all.

A sibling: a Code node that failed to set `skip: false` on a field a downstream strict IF then
compared — strict validation throws on `undefined` rather than treating it as falsy.

**Takeaway:** error-handling branches need a test that actually forces the error. An untriggered
branch is an untested branch.

---

## Class 5 — Data flow through Config nodes

A structural pattern that broke four workflows at once, twice.

**Config Set nodes dropping the payload.** Every workflow begins with a `⚙️ Client Config` Set node.
Without `includeOtherFields: true`, a Set node emits *only* its own fields — silently discarding the
webhook body or trigger payload. Eight config nodes had this defect. Downstream, availability checks
defaulted to today and bookings were built from empty data.

**`$input.first()` after a Config node.** Code nodes reading the trigger payload via `$input` get
whatever the *previous* node emitted, not the original trigger. The fix is to reference the trigger
explicitly: `$('Execute Workflow Trigger').first().json`.

**Takeaway:** in n8n, "the data" is always relative to the node asking. Reference the source by name.

---

## Class 6 — Concurrency

Discovered only under deliberate simultaneous load, never in sequential testing.

**Positional append race.** Sheets appends allocated row positions client-side. Two simultaneous
bookings could target the same row. Fixed by POSTing to Google's `values:append` endpoint, which
allocates server-side, and by reading the live header row and mapping by column *name* so reordering
a column cannot misalign a write. Verified with 8 simultaneous escalations (8/8 rows) and 2
simultaneous bookings (2/2 complete).

**Hollow rows.** Fixing the above exposed a worse variant: a two-step write (core fields, then
first-touch fields) produced a *partial* record when two new leads arrived together — phone number
and first-touch data, no name, no email, no status. Worse than losing the row, because it looks like
a real record. Collapsed to a single read → merge → atomic write.

**Session race measured in seconds, not milliseconds.** Chatbot session state was assumed to have a
millisecond-wide race window. Measurement showed the window is the *entire reply generation time* —
4.6–9.6s for a direct reply, 7.8–15.8s when a tool runs. Three messages sent one second apart lost
the first two, every time. That is the standard human opening: "hi" / "how are you" / the actual
question. Fixed by reloading and merging session state immediately before each save. Verified 3/3 at
1s gaps, up from 1/3.

**Takeaway:** the race window is however long your slowest path takes, and users type faster than
an LLM answers.

---

## Class 7 — Third-party limits

**Twilio's 1,600-character cap.** Replies over the limit were rejected outright with error `21617`
and the customer received *nothing* — not a truncated message, no message. Fixed by chunking at
paragraph boundaries under 1,500 characters, capped at 3 messages.

**Retry on a non-idempotent endpoint.** VAPI's `POST /call` places a real phone call. `retryOnFail`
on that node means a transient network error results in the customer being **called twice**. Those
nodes are deliberately `retryOnFail: false` — the one place in the codebase where retry is wrong,
documented so a future consistency sweep doesn't "fix" it.

**A documented timeout that was never verified.** The build rules asserted a 5-second VAPI tool
timeout for two months. It was never checked against VAPI's documentation; the real setting is
`server.timeoutSeconds`, defaulting to 20. The invented figure drove real architecture decisions,
including deferring a fix. Corrected, with the error itself left in the record.

**Takeaway:** write down where each external constraint came from. An unsourced number becomes fact.

---

## Class 8 — Reference integrity

The worst single defect in the project, and the one that best explains why static validation cannot
be relied on.

Five nodes referenced the config node as an **escaped literal**:

```javascript
// What five nodes actually contained (an escaped literal, 22 chars):
$('\\u2699\\ufe0f Client Config')

// What n8n needed to match the node by name:
$('⚙️ Client Config')
```

n8n matches node names by exact string. All five threw on **every execution**. Four were Google
Calendar nodes — check availability, book, reschedule, cancel — plus the message logger. Because
that dispatcher is shared by the outbound agent, the inbound agent and the reactivation engine,
**booking was dead across the entire product.**

The strict validator reported 0 errors and passed all 115 expressions, correctly: `$('anything')`
is valid syntax. The defect only surfaced because the validator quoted the raw string inside a
`cachedResultName` warning that had been dismissed as a false positive for three days.

Likely cause: an MCP `updateNode` call that passed a double-escaped `"\\u2699"`. This produced a
standing rule — *after any update writing an expression that contains the config node name, re-read
the node and confirm the literal `⚙️` came back. An API `success` response proves nothing about what
was stored.*

**Takeaway:** verify writes by reading them back. Especially through an API, especially with
non-ASCII identifiers.

---

## Class 9 — Security

Found in a dedicated pass, after the systems were otherwise considered finished.

**All four voice-agent webhooks were publicly unauthenticated**, as were the chatbot's. The worst
was `/lead-intake` — an open, unauthenticated endpoint that **places real outbound phone calls.**
Anyone who discovered the URL could make the system dial arbitrary numbers, at the account owner's
expense, from the client's caller ID.

Others allowed forging an inbound message to book, cancel, or opt out **as any customer**, given
only their phone number.

All now authenticated: header-secret auth for the VAPI and intake endpoints, basic auth for the
Twilio inbound handler.

Separately, the **VAPI API key was stored in plaintext inside Set nodes**, duplicated across six
config nodes, and therefore present in every workflow export. Moved into an n8n credential.

**Takeaway:** a webhook with no auth is a public API. Ask what the worst caller could do with it —
here, the answer was "spend the client's money making calls in their name."

---

## Summary

| Class | Root cause | Catchable by static validation? |
|---|---|---|
| 1 Silent write failures | Column names, type coercion | No |
| 2 Timezone / dates | Runtime env + JS `Date` semantics | No |
| 3 Phantom success | Unconditional success paths | No |
| 4 Unreachable handling | Strict type comparison semantics | No |
| 5 Config data flow | Node-relative data model | No |
| 6 Concurrency | Timing under simultaneous load | No |
| 7 Third-party limits | Undocumented external constraints | No |
| 8 Reference integrity | String-exact node lookup | No — syntactically valid |
| 9 Security | Missing auth, plaintext secrets | No |

Nine classes, one conclusion: **the validator tells you a workflow is well-formed, not that it does
what you intended.** The methods that did find these are in
[`audit-methodology.md`](audit-methodology.md).

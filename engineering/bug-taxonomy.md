# Bug Taxonomy

111 bug IDs were logged across the three largest systems between April and August 2026. This page
groups them into the patterns that kept recurring.

One finding sits behind all of it:

> **Static validation caught none of them.**

n8n's validator ran in `strict` profile on every workflow, repeatedly, and kept reporting 0 errors.
It checks structure: node types exist, required parameters are present, connections point at real
nodes, expressions parse. Everything below is structurally valid and behaviourally wrong.

## Class 1: silent write failures

The most dangerous group, because the workflow reports success.

**`Cancelled At` versus `CancelledAt`.** Three workflows wrote a cancellation timestamp to a column
with a space in the name. The live sheet's column has no space. Google Sheets does not error on an
unmatched column name in an `appendOrUpdate`, it just doesn't write that field. Every cancellation
in the voice agent recorded successfully and lost its timestamp. Found by diffing every column name
in every Sheets node against the schema file, not by testing.

**Luxon objects where strings were expected.** `{{ $now }}` resolves to a Luxon `DateTime` object,
not a string. Written into a cell it produces unusable output. Found in five separate nodes. In four
of those cases, three sibling nodes had already been fixed and one was missed, a pattern common
enough that it got its own audit rule.

**`cellFormat: RAW`.** 36 Sheets writes stored values raw, which can coerce a phone number into a
number and lose the leading `+`. That matters for compliance, not just tidiness: the opt-out check
downstream is an exact phone-string match, so a reformatted number stops matching its own opt-out
record.

The lesson: an external system accepting your write is not confirmation it stored what you meant.

## Class 2: timezone and date arithmetic

**`setHours()` on a UTC server.** Booking logic used `setHours()` to build appointment times. n8n
Cloud runs UTC and the business runs Europe/London. From late March to late October the UK is on
BST, so every booking made during British Summer Time was written an hour late, all summer. Fixed
with an explicit offset (`month 3 to 10 gives +1, otherwise +0`) applied the same way everywhere.

**V8 defaults yearless date strings to 2001.** A parser took "the 14th of March" and produced
`2001-03-14`, which is in the past, so availability checks came back with nothing bookable. That is
JavaScript `Date` behaviour rather than an n8n quirk, and it is invisible unless you look at the
parsed value.

**Explicit day and month discarded.** A related parser preferred a relative reading and threw away
the date the customer had actually given.

Timezone bugs are seasonal. A build tested in February is not tested for BST.

## Class 3: phantom success

Branches that report success without checking whether anything succeeded.

Five places returned a success message unconditionally. Two mattered a lot: the do-not-contact
confirmation was spoken aloud to a caller before the opt-out had been verified as written. Under UK
GDPR and PECR, telling someone they have been opted out when they haven't is a compliance failure,
not a cosmetic one.

Also here: failed reminders and follow-ups marked as sent and therefore never retried, and a
calendar read failure that reported the whole day as free, offering slots that were already booked.

A `try/catch` that swallows the error and returns the success path is worse than no handler, because
it destroys the evidence.

## Class 4: unreachable error handling

Three IF nodes existed to catch calendar failures. Under n8n's strict `typeValidation` the
comparison they used could never be true, so the error branch was dead code. The workflow looked
carefully defensive and had no defence at all.

Related: a Code node that didn't set `skip: false` on a field a downstream strict IF then compared.
Strict validation throws on `undefined` rather than treating it as falsy.

An error branch nothing has ever triggered is an untested branch.

## Class 5: data flow through config nodes

A structural pattern that broke four workflows at once, twice.

**Config Set nodes dropping the payload.** Every workflow starts with a `⚙️ Client Config` Set node.
Without `includeOtherFields: true` a Set node emits only its own fields and silently discards the
webhook body. Eight config nodes had this. Downstream, availability checks defaulted to today and
bookings were built from empty data.

**`$input.first()` after a config node.** Code nodes reading the trigger payload through `$input`
get whatever the previous node emitted, not the original trigger. The fix is to name the source:
`$('Execute Workflow Trigger').first().json`.

In n8n, "the data" is always relative to the node asking for it.

## Class 6: concurrency

Only found under deliberate simultaneous load. Sequential testing passes every time.

**Positional append race.** Sheets appends allocated row positions client side, so two simultaneous
bookings could target the same row. Fixed by POSTing to Google's `values:append`, which allocates
server side, and by reading the live header row and mapping by column name so reordering a column
can't misalign a write. Verified with eight simultaneous escalations (eight rows) and two
simultaneous bookings (two complete rows).

**Half-empty rows.** Fixing that exposed something worse. A two-step write (core fields, then
first-touch fields) produced a partial record when two new leads arrived together: phone number and
first-touch data, no name, no email, no status. Worse than losing the row, because it looks real.
Collapsed into one read, one merge, one write.

**A race measured in seconds, not milliseconds.** Chatbot session state was assumed to have a
millisecond-wide window. Measuring it showed the window is the entire reply time: 4.6 to 9.6 seconds
for a direct reply, 7.8 to 15.8 with a tool. Three messages a second apart lost the first two every
time, which is the standard human opening of "hi", "how are you", then the real question. Fixed by
reloading and merging session state right before each save. Verified three out of three at one
second gaps, up from one out of three.

The race window is however long your slowest path takes, and people type faster than an LLM answers.

## Class 7: third-party limits

**Twilio's 1,600 character cap.** Longer replies were rejected outright with error `21617` and the
customer received nothing. Not a truncated message, nothing. Fixed by chunking at paragraph
boundaries under 1,500 characters, capped at three messages.

**Retry on a non-idempotent endpoint.** VAPI's `POST /call` places a real phone call. `retryOnFail`
there means a transient network error calls the customer twice. Those nodes are deliberately
`retryOnFail: false`, documented so a later consistency sweep doesn't "fix" it.

**A documented constant nobody had checked.** The build rules asserted a 5-second VAPI tool timeout
for two months. It had never been checked against VAPI's documentation. The real setting is
`server.timeoutSeconds`, defaulting to 20. The invented number drove real architecture decisions,
including deferring a fix that didn't need deferring. Corrected, with the mistake left in the record.

Write down where each external constraint came from. An unsourced number turns into a fact.

## Class 8: reference integrity

The worst single bug in the project, and the best argument against trusting static validation.

Five nodes referenced the config node as an escaped literal:

```javascript
// What five nodes actually contained, an escaped literal, 22 characters:
$('\\u2699\\ufe0f Client Config')

// What n8n needed to match the node by name:
$('⚙️ Client Config')
```

n8n matches node names by exact string. All five threw on every execution. Four were Google Calendar
nodes (check availability, book, reschedule, cancel) plus the message logger. Because that
dispatcher is shared by the outbound agent, the inbound agent and the reactivation engine, booking
was dead across the entire product.

The strict validator reported 0 errors and passed all 115 expressions, correctly. `$('anything')` is
valid syntax. It surfaced only because the validator quoted the raw string inside a
`cachedResultName` warning I had dismissed as a false positive for three days.

Likely cause: an MCP `updateNode` call passing a double-escaped `"\\u2699"`. The rule that came out
of it is that after any update writing an expression containing the config node name, re-read the
node and confirm the literal `⚙️` came back. An API `success` response proves nothing about what got
stored.

Verify writes by reading them back, especially through an API and especially with non-ASCII
identifiers.

## Class 9: security

Found in a dedicated pass, after the systems were otherwise considered finished.

All four voice agent webhooks were publicly unauthenticated, as were the chatbot's. The worst was
`/lead-intake`, an open endpoint that places real outbound phone calls. Anyone who found the URL
could make the system dial arbitrary numbers, at the owner's expense, from the client's caller ID.

The others allowed forging an inbound message to book, cancel or opt out as any customer, given only
their phone number.

All are authenticated now: header secrets for the VAPI and intake endpoints, basic auth for the
Twilio inbound handler.

Separately, the VAPI API key was sitting in plaintext inside Set nodes, duplicated across six config
nodes, and therefore present in every workflow export. Moved into an n8n credential.

A webhook with no auth is a public API. Ask what the worst caller could do with it. Here the answer
was "spend the client's money making calls in their name".

## Summary

| Class | Root cause | Catchable by static validation? |
|---|---|---|
| 1 Silent write failures | Column names, type coercion | No |
| 2 Timezone and dates | Runtime environment, JS `Date` semantics | No |
| 3 Phantom success | Unconditional success paths | No |
| 4 Unreachable handling | Strict type comparison semantics | No |
| 5 Config data flow | Node-relative data model | No |
| 6 Concurrency | Timing under simultaneous load | No |
| 7 Third-party limits | Undocumented external constraints | No |
| 8 Reference integrity | String-exact node lookup | No, it is valid syntax |
| 9 Security | Missing auth, plaintext secrets | No |

Nine classes, one conclusion. The validator tells you a workflow is well formed. It does not tell
you it does what you meant. What actually found these is in [`audit-methodology.md`](audit-methodology.md).

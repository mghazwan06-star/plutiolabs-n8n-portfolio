# Build Standards

The conventions I use across all 40 workflows. Each one exists because something broke, and I cite
the bug class so the rule can be re-examined rather than copied blindly.

## Structure

**Every workflow opens with `⚙️ Client Config`.** A Set node right after the trigger holding every
client-specific value: business name, phone numbers, calendar ID, spreadsheet ID, business hours,
timezone offset. Nothing client-specific lives anywhere else. This is what makes a workflow reusable
between clients, because deploying for a new one means editing a single node.

**`includeOtherFields: true` on every config Set node.** Without it, a Set node emits only its own
fields and silently drops the trigger payload. (Class 5)

**Sticky notes are required.** One overview note per workflow, plus one per major section, 56 in
total. Documentation next to the nodes it describes cannot drift the way an external document does,
and it survives export, which is why the diagrams in this repo can render it.

## Code nodes

**ES5 only.** `var` and `function()`. No `const`, `let`, arrow functions or optional chaining. Some
n8n Code node runtimes reject newer syntax and the failure is confusing. 140 Code nodes, roughly 7,200 lines, all ES5.

**`try/catch` in every Code node**, with `onError` set on the node. The catch returns a readable
message, never a raw stack trace, and above all never one to a voice agent that will read it aloud
to a customer.

**A catch must not claim success.** The catch path reports failure. (Class 3)

**Name the trigger, don't use `$input`.** Use `$('Execute Workflow Trigger').first().json`. `$input`
gives you whatever the previous node emitted. (Class 5)

**Use `.first()`, not `.item`,** when referencing another node in "All Items" mode.

## Expressions

**Webhook data is under `$json.body`,** not `$json`. The single most common mistake when writing n8n
webhook workflows.

**Never write a Luxon object into a cell.** `{{ $now }}` is a `DateTime` object. Format it. (Class 1)

**Explicit BST offset, never `setHours()`.** The server runs UTC, the business runs Europe/London:
month 3 to 10 gives +1, otherwise +0. (Class 2)

## Node properties the validator does not check

| Property | Applies to | Why |
|---|---|---|
| `onError` | every Code node | One bad item otherwise aborts the run |
| `alwaysOutputData` | every Sheets read | An empty read otherwise halts the branch |
| `retryOnFail` | Sheets, Calendar | Transient Google API failures are routine |
| `retryOnFail: false` | VAPI `POST /call` | Not idempotent. A retry places a second real call (Class 7) |
| `fallbackOutput` | every Switch | Unmatched input otherwise disappears |
| `range: "A:Z"` | every Sheets read | Reads fail without an explicit range |
| `includeOtherFields` | every config Set | Payload is dropped without it (Class 5) |

## Data integrity

**Column names come from one schema document,** and I diff every Sheets write against it. (Class 1)

**Match rows on a stable key, not a phone number.** `CalendarEventId` for appointments. Phone
matching updates the wrong row as soon as a customer has two records.

**Atomic appends.** Row allocation happens server side through Google's `values:append`, and the
live header row is read so fields map by name. Reordering a column cannot misalign a write. (Class 6)

**One write per record.** Assemble the record fully in memory, then write once. Two-step writes
produce half-empty rows under concurrency. (Class 6)

## Security

**No credentials in node parameters.** API keys go in n8n credentials, never in Set nodes, because
Set nodes end up in every workflow export. (Class 9)

**Every webhook is authenticated.** Header secret or basic auth. A public webhook is a public API.
(Class 9)

**Assume the payload is hostile.** A webhook that takes a phone number and acts on that customer's
record has to authenticate the caller, not trust the number.

## Verification

**Re-read after write.** After any API update that writes an expression containing a node name,
fetch the node back and check the stored value. `success` describes the request, not the result.
(Class 8)

**Validate before and after deployment.** Necessary, not sufficient. See
[`bug-taxonomy.md`](bug-taxonomy.md) for what it misses.

**Prefer partial updates to full replacement.** Full-workflow replacement has silently wiped Switch
node rules during bulk operations.

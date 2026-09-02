# Build Standards

The conventions enforced across all 41 workflows. Each exists because something broke — the
originating defect class is cited so the rule can be re-evaluated rather than cargo-culted.

## Structure

**Every workflow opens with `⚙️ Client Config`.** A Set node immediately after the trigger holding
every client-specific value: business name, phone numbers, calendar ID, spreadsheet ID, business
hours, timezone offset. Nothing client-specific appears anywhere else. This is what makes a
workflow reusable across clients — deploying for a new client means editing one node.

**`includeOtherFields: true` on every config Set node.** Without it a Set node emits only its own
fields and silently discards the trigger payload. *(Class 5)*

**Sticky notes are mandatory.** One workflow-overview note, plus one per major section. 57 sticky
notes across the repository. Documentation beside the nodes it describes cannot drift out of sync
the way an external document does — and it survives export, which is why the diagrams in this
repository can render it.

## Code nodes

**ES5 only** — `var`, `function()`. No `const`, `let`, arrow functions, or optional chaining. Some
n8n Code node runtimes reject newer syntax, and the failure mode is confusing. 157 Code nodes,
~7,500 lines, uniformly ES5.

**`try/catch` in every Code node**, with `onError` set at the node level. The catch must return a
graceful, human-readable message — never a raw stack trace, and above all never to a voice agent
that will read it aloud to a customer.

**A catch must not fabricate success.** The catch path reports failure. *(Class 3)*

**Reference the trigger by name, never `$input`.** Use `$('Execute Workflow Trigger').first().json`.
`$input` yields whatever the immediately preceding node emitted. *(Class 5)*

**Use `.first()`, not `.item`,** when referencing another node in "All Items" mode.

## Expressions

**Webhook data lives under `$json.body`.** Not `$json`. The most common single mistake when writing
n8n webhook workflows.

**Never write a Luxon object into a cell.** `{{ $now }}` is a `DateTime` object; format it
explicitly. *(Class 1)*

**Explicit BST offset, never `setHours()`.** The server runs UTC, the business runs Europe/London:
`month 3–10 → +1, else +0`. *(Class 2)*

## Node properties the validator does not check

| Property | Applies to | Why |
|---|---|---|
| `onError` | every Code node | Otherwise one bad item aborts the run |
| `alwaysOutputData` | every Sheets read | An empty read otherwise halts the branch |
| `retryOnFail` | Sheets, Calendar | Transient Google API failures are routine |
| `retryOnFail: false` | VAPI `POST /call` | Non-idempotent — a retry places a second real call *(Class 7)* |
| `fallbackOutput` | every Switch | Unmatched input otherwise vanishes silently |
| `range: "A:Z"` | every Sheets read | Reads fail without an explicit range |
| `includeOtherFields` | every config Set | Payload is dropped without it *(Class 5)* |

## Data integrity

**Column names come from a single canonical schema document.** Every Sheets write is diffed against
it. *(Class 1)*

**Match rows on a stable unique key**, not on phone number — `CalendarEventId` for appointments.
Phone-number matching updates the wrong row when a customer has two records.

**Atomic appends.** Row allocation happens server-side via Google's `values:append`; the live header
row is read and fields mapped by *name*, so reordering a column cannot misalign a write. *(Class 6)*

**One write per record.** A record is assembled fully in memory, then written once. Two-step writes
produce hollow rows under concurrency. *(Class 6)*

## Security

**No credentials in node parameters.** API keys live in n8n credentials, never in Set nodes — they
end up in every workflow export otherwise. *(Class 9)*

**Every webhook is authenticated.** Header secret, or basic auth. A public webhook is a public API.
*(Class 9)*

**Assume the payload is hostile.** A webhook that accepts a phone number and acts on that customer's
record must authenticate the *caller*, not trust the number.

## Verification

**Re-read after write.** After any API update that writes an expression containing a node name,
fetch the node back and confirm the stored value. `success` describes the request, not the result.
*(Class 8)*

**Validate before and after deployment** — necessary, not sufficient. See
[`bug-taxonomy.md`](bug-taxonomy.md) for what it does not catch.

**Prefer partial updates to full replacement.** Full-workflow replacement has silently wiped Switch
node rules during bulk operations.

# Production n8n Automation Systems

Five connected automation systems for UK home-service businesses, built on n8n with Claude, VAPI,
Twilio, Google Workspace and Apify. **41 workflows, 616 nodes, roughly 7,500 lines of JavaScript**,
exported from a live instance and documented here.

Every diagram is generated from the workflow's own canvas coordinates. Nothing is redrawn or mocked up.

```mermaid
flowchart LR
  LP[Landing page<br/>form]:::e --> D2
  PH[Inbound<br/>phone call]:::e --> D2
  WA[WhatsApp<br/>message]:::e --> D3
  D4[<b>Lead Scraper</b><br/>3 workflows]:::s --> CRM
  CRM -.dormant leads.-> D5
  D2[<b>Voice Agent</b><br/>10 workflows, 262 nodes]:::s --> TD
  D3[<b>WhatsApp Chatbot</b><br/>10 workflows, 121 nodes]:::s --> TD
  D5[<b>Lead Reactivation</b><br/>7 workflows, 94 nodes]:::s --> TD
  TD[<b>Shared Tool Dispatcher</b><br/>56 nodes, 8 tools]:::c
  TD --> CAL[(Google<br/>Calendar)]:::d
  TD --> CRM[(Sheets CRM<br/>34 columns)]:::d
  D2 --> CRM
  D3 --> CRM
classDef e fill:#fff4e5,stroke:#f0a30a,color:#3d2f00
classDef s fill:#e8f1fe,stroke:#408fdd,color:#0b2545
classDef c fill:#f3ecff,stroke:#8b5cf6,color:#2c1a4d
classDef d fill:#e6f7ee,stroke:#0f9d58,color:#04331f
```

Three systems share one booking dispatcher. That removed three copies of calendar logic, and it
concentrated the blast radius. Both halves of that show up below.

## The systems

| System | Workflows | Nodes | What it does |
|---|---:|---:|---|
| [**AI Voice Agent**](systems/01-voice-agent.md) | 10 | 262 | Answers inbound calls 24/7, rings new leads within minutes, books into Google Calendar, recovers missed calls, sends reminders |
| [**WhatsApp Chatbot**](systems/02-whatsapp-chatbot.md) | 10 | 121 | Claude-powered receptionist with session memory and seven tool workflows. Books, reschedules and cancels inside the chat |
| [**Lead Reactivation**](systems/03-lead-reactivation.md) | 7 | 94 | Scans a dormant lead database and runs AI voice re-engagement campaigns with cadence control |
| [**Lead Scraper v4**](systems/04-lead-scraper.md) | 3 (+10 utils) | 58 | Parent/child batch pipeline: scrape, enrich, score and route into a 34-column CRM |
| [**AI Ad Engine**](systems/05-ad-engine.md) | 1 | 38 | Weekly batch producing ad copy and image prompts into a human review queue |

## The part worth reading

Over eight months, 17 audit passes logged 111 bugs.

**Static validation caught none of them.**

That is not a complaint about the validator. It does what it says: checks that nodes exist,
parameters are filled in, connections resolve and expressions parse. Here is what that leaves out.

### The bug that killed booking in three systems

Five nodes referenced the config node like this:

```javascript
// What was actually stored in five nodes, an escaped literal, 22 characters:
$('\\u2699\\ufe0f Client Config')

// What n8n needed in order to match the node by name:
$('⚙️ Client Config')
```

n8n matches node names by exact string, so all five threw on every execution. Four of them were the
Google Calendar nodes: check availability, book, reschedule, cancel. Because that dispatcher is
shared by the voice agent, the inbound caller and the reactivation engine, booking was dead across
the whole product.

The strict validator reported 0 errors and passed all 115 expressions, correctly. `$('anything')`
is valid syntax. It only surfaced because the validator quoted the raw string inside a warning I had
been dismissing as a false positive for three days.

The likely cause was an API call that passed a double-escaped `"\\u2699"`. The rule that came out of
it: after writing any expression containing a node name, read it back. An API `success` response
tells you about the request, not about what got stored.

### Eight more

| Bug | What it did |
|---|---|
| `setHours()` on a UTC server | Put every booking an hour late for a whole summer (BST) |
| Column written as `Cancelled At`, live column is `CancelledAt` | Silently dropped every cancellation timestamp |
| Unconditional success in five response builders | Voice agent told callers they were opted out before the write was checked |
| Strict `typeValidation` on three calendar-failure gates | Made the error branches unreachable. Defensive code that never ran |
| Config Set nodes missing `includeOtherFields` | Dropped the trigger payload, so bookings were built from empty data |
| Client-side row allocation on Sheets append | Simultaneous bookings collided. The fix then exposed half-empty rows, which are worse because they look real |
| Session state read and written across a 5 to 16 second window | Three messages a second apart lost the first two, which is the normal way people open a chat |
| Twilio's 1,600 character cap | Long replies were rejected outright and the customer received nothing at all |

Full write-up, nine classes with root causes: **[`engineering/bug-taxonomy.md`](engineering/bug-taxonomy.md)**

### And one found while building this repo

Sanitizing the exports for publication turned up two live API keys hardcoded in plaintext in the Ad
Engine's config node, an OpenAI key and a Perplexity key. They would have been published verbatim.
Then GitHub's own secret scanning blocked the first push over a Twilio Account SID that my sanitizer
had no pattern for. Two scanners, two different misses. Both are written up rather than quietly
cleaned up.

## What the workflows look like

Each diagram renders the workflow's real layout, including the sticky notes carrying its design notes.

![handle_dnc workflow](assets/diagrams/wf-c3f-handle-dnc.svg)

The `handle_dnc` opt-out tool. Its on-canvas note lists every CRM field it writes and why, sitting
next to the nodes that write them.

## Engineering documentation

| | |
|---|---|
| [**Bug taxonomy**](engineering/bug-taxonomy.md) | 111 bugs, nine classes, root causes, and why each one was invisible to validation |
| [**Audit methodology**](engineering/audit-methodology.md) | 17 passes ranked by what each kind actually caught, and what I would do differently |
| [**Build standards**](engineering/build-standards.md) | Every convention used across the 41 workflows, each traced back to the bug that caused it |
| [**How Claude was used**](engineering/how-claude-was-used.md) | Claude Code and an n8n MCP server as the build environment, including where it failed |

## Start here

- **Two minutes:** this page, then the [voice agent](systems/01-voice-agent.md) architecture diagram.
- **Ten minutes:** [`bug-taxonomy.md`](engineering/bug-taxonomy.md), which is where the engineering is.
- **Hands on:** import any export from [`workflows/`](workflows/). Credentials are stubbed, structure is intact.

## Business context

Built for UK home-service installers: solar, HVAC, insulation, heating. Per ONS, 94.8% of firms in
this market have fewer than ten employees. There is no operations team, so the owner answers the
phone between jobs and quotes get chased in the evening or not at all.

The goal is not more leads. UK installers are mostly demand-constrained, and the real bottleneck is
the owner's admin time. These systems go after response speed and follow-up consistency instead,
which is what converts the demand already there. One commitment runs through all of them: a captured
lead gets an automated response in under two minutes, at any hour, with nobody involved.

Client names, pricing and commercial terms are left out. Credentials appear as `CREDENTIAL_ID`,
account identifiers as `YOUR_*`, keys as `*_REDACTED`.

**Stack:** n8n, Claude (Anthropic API), VAPI, Twilio, Google Sheets and Calendar, Supabase, Apify

# Screenshots

The diagrams in this repository are generated from each workflow's real canvas coordinates, so the
architecture is fully documented without screenshots. What screenshots add is **evidence that the
systems actually ran** — which generated diagrams cannot show.

Drop files here and reference them from the relevant page. Suggested set, in order of value:

| File | Shows | Why it matters |
|---|---|---|
| `execution-history.png` | n8n executions list with successful runs | The single most convincing image available — proves real execution, not just a valid build |
| `wf03-canvas.png` | WF3 Tool Dispatcher in the n8n editor | 57 nodes, 8 tools, shared by three systems |
| `wf06-canvas.png` | WF6 Inbound SMS Handler | 51 nodes of intent routing |
| `c2-canvas.png` | WF-C2 Claude Core | The two-round tool-calling loop |
| `whatsapp-conversation.png` | A real chat with the bot | Booking completed inside the conversation |
| `crm-sheet.png` | The Sheets CRM with rows | Structured output of the pipeline |
| `vapi-transcript.png` | A VAPI call transcript | The voice agent qualifying and booking |
| `scraper-run.png` | A completed scraper batch | The pipeline at work |

## ⚠️ Redact before adding

Screenshots leak more than sanitized JSON does. Before committing any image, remove:

- **Customer names, phone numbers, email addresses** — execution logs and CRM rows are full of them
- **The n8n instance URL** — visible in the browser address bar on every editor screenshot
- **Webhook URLs** — visible in webhook node panels
- **API keys and credential names** — visible in credential dropdowns and node parameter panels
- **Client business names** — this repository is otherwise client-anonymous

Blur or crop rather than relying on a small font size. A 4K screenshot is readable when zoomed.

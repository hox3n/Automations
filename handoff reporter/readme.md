An n8n workflow that turns a client's sales-call notes into a formatted **Sales Handoff Report** in Google Docs and posts the link to Slack.

It is triggered externally (from Make) with a client ID and a Slack channel ID, and runs unattended end to end — no human step between the trigger and the Slack message.

---

## What it produces

A Google Doc titled `Handoff Report - <client name>`, created in a fixed Drive folder, shared read-only with the agency domain, containing these sections:

**Sales Conversations**
- Their Goals
- Biggest Problems
- What we said we'd do
- What they want from us
- What was discussed

**Moving Forward**
- Key talking points
- Unclosed Loops from the Sales Process

If none of the notes contain sales content, the report says so instead of inventing material.

---

## The active path

```
Webhook
  → HTTP Request1        fetch contact notes from GoHighLevel
  → Filter in Call notes keep only notes containing "Attendees"
  → Split Out            one item per note
  → Aggregate1           collect all note bodies
  → OpenAI               generate the report as markdown
  → Get Content          pull the message text into `content`
  → Create Google Docs   create the doc in the Drive folder
  → Convert Markdown…    translate markdown into Docs API requests
  → Update Google Docs…  batchUpdate the doc with the formatted content
  → drive permissions    share read-only with the agency domain
  → Slack                post the doc link to the channel
```

### Step detail

**1. Webhook** — `POST` trigger. Expects a body with:

| Field | Used for |
|---|---|
| `client_id` | GoHighLevel contact lookup |
| `channel_id` | which Slack channel to post to |
| `client_name` | the document title |
| `opportunity_id` | present in the payload, not currently read |

**2. HTTP Request1** — `GET /v1/contacts/{client_id}/notes/` against the GoHighLevel REST API, with a bearer token set directly on the node as a header.

**3. Filter in Call notes** — a Code node that keeps only notes whose body contains the string `Attendees`. This is the heuristic for "this note is a call recap rather than a manual note."

**4. Split Out → Aggregate1** — splits the `notes` array into individual items, then re-aggregates the `body` fields so the model receives all notes in a single prompt.

**5. OpenAI** — `gpt-4o-mini`. The system prompt pins the exact markdown skeleton and forbids anything outside it: no preamble, no extra sections, no decorative symbols. Bullet counts per section are left to the content.

**6. Get Content** — a Set node that lifts `message.content` into a plain `content` field, so later nodes don't have to reach into the OpenAI response shape.

**7. Create Google Docs** — creates an empty doc named `Handoff Report - {{ client_name }}` in a hard-coded Drive folder.

**8. Convert Markdown to Requests** — the most substantial Code node. Google Docs' API has no markdown import, so this hand-rolls a converter that walks the markdown line by line and emits Docs API requests:

- `#`–`######` → `insertText` + `updateParagraphStyle` with `HEADING_1`…`HEADING_6`
- `- ` / `* ` → `insertText` + `createParagraphBullets` (disc/circle/square preset)
- `1. ` → `insertText` + `createParagraphBullets` (decimal/alpha/roman preset)
- `**bold**` / `__bold__` → `updateTextStyle { bold: true }`, with the markers stripped
- `*italic*` / `_italic_` → `updateTextStyle { italic: true }`

It tracks a running `currentIndex` as it inserts, because every insertion shifts the positions of everything after it. It outputs the request array, the document ID, and the `batchUpdate` URL to call.

**9. Update Google Docs File** — `POST`s that request array to `https://docs.googleapis.com/v1/documents/{id}:batchUpdate`, which is what actually fills the document.

**10. drive permissions** — `POST`s a Drive permission granting `reader` to the whole domain, so anyone can open the link without being added individually.

**11. Slack** — posts `https://docs.google.com/document/d/{id}/edit?usp=sharing` to the channel from the webhook payload.

---

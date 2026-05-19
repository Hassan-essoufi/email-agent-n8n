# Architecture

This document describes every node in the n8n workflow and how they connect to form the email automation pipeline.

---

## High-Level Flow

```
Gmail (unread emails)
        │
        ▼
  Loop Over Items ──────────────────────────────────────┐
        │                                               │
        ▼                                               │
   Edit Fields                                         │
        │                                               │
        ▼                                               │
   AI Summary (Groq)                                   │
        │                                               │
        ▼                                               │
  AI Classifier (Groq)                                 │
        │                                               │
        ▼                                               │
      Switch                                           │
     /      \                                          │
event        sheet                                    │
  │             │                                      │
  ▼             ▼                                      │
Event        Append row                               │
Extractor    in Google Sheets                         │
  │                │                                   │
  ▼                └──────────────────┐               │
Code (JSON parse)                     │               │
  │                                   ▼               │
  ▼                           Remove label (mark read) │
Create Calendar event ────────────────┘               │
                                      │               │
                                      └───────────────┘
                                      (loop continues)
```

---

## Node Reference

### Schedule Trigger

**Type:** `n8n-nodes-base.scheduleTrigger`

Fires the workflow on a fixed interval (default: every 1 minute). This is the entry point for the entire pipeline. No data is passed into the first node — the trigger simply wakes up the workflow.

---

### Get many messages

**Type:** `n8n-nodes-base.gmail` (operation: `getAll`)

Connects to Gmail via OAuth 2.0 and fetches up to **10 unread messages** per run using the search filter `is:unread`. The node returns a simplified array of message objects containing fields like `id`, `threadId`, `From`, `Subject`, `To`, `snippet`, `labels`, and `internalDate`.

The limit of 10 is intentional — it keeps each workflow execution short and avoids Gmail API rate limits. Subsequent runs pick up the next batch.

---

### Loop Over Items

**Type:** `n8n-nodes-base.splitInBatches`

Iterates over the array of emails one item at a time. After each email is fully processed (summarised, classified, routed, marked as read), the loop feeds the next item into the pipeline. When all items are exhausted, the loop exits and the workflow execution ends.

---

### Edit Fields

**Type:** `n8n-nodes-base.set` (mode: raw JSON)

Transforms the raw Gmail message object into a clean, flat JSON structure used by every downstream node:

| Field | Source |
|---|---|
| `email_id` | `$json.id` |
| `thread_id` | `$json.threadId` |
| `from` | `$json.From` |
| `subject` | `$json.Subject` |
| `to` | `$json.To` |
| `snippet` | `$json.snippet` |
| `labels` | `$json.labels[].name` joined as string |
| `received_at` | `new Date(Number($json.internalDate)).toISOString()` |
| `email_content` | Formatted multi-line string combining all fields above |

The `email_content` field is the full text passed to AI prompts.

---

### AI Summary

**Type:** `@n8n/n8n-nodes-langchain.agent`  
**LLM:** Groq — `openai/gpt-oss-20b` (via Groq Chat Model node)

Reads the structured email fields and returns a **2–3 sentence plain-text summary** covering:
- What the email is about
- Who sent it
- What action (if any) is required

The summary is stored in `$json.output` and later written to Google Sheets.

---

### AI Classifier

**Type:** `@n8n/n8n-nodes-langchain.agent`  
**LLM:** Groq — `openai/gpt-oss-20b` (via Groq Chat Model2 node)

Receives the AI summary output and classifies the email into **exactly one** of two categories:

- `event` — email contains a meeting, interview, appointment, call, schedule, or calendar invitation
- `sheet` — everything else (newsletters, job alerts, promotions, notifications, invoices, etc.)

The node outputs only the single word `event` or `sheet` with no extra text.

---

### Switch

**Type:** `n8n-nodes-base.switch` (mode: Rules)

Routes the item based on the classifier output:

- Output 0 → `event` path
- Output 1 → `sheet` path

Uses strict string equality matching on `$json.output`.

---

### Event Extractor *(event path)*

**Type:** `@n8n/n8n-nodes-langchain.agent`  
**LLM:** Groq — `openai/gpt-oss-120b` (via Groq Chat Model1 node, higher capacity for structured extraction)

Reads `email_content` and extracts structured calendar data as a JSON object:

```json
{
  "title": "Meeting title",
  "start": "2026-05-22T14:00:00+01:00",
  "end":   "2026-05-22T15:00:00+01:00",
  "description": "Brief description of the event"
}
```

All datetimes use **ISO 8601 format** with the Morocco timezone offset (`+01:00`). If no end time is found in the email, it defaults to start + 1 hour.

---

### Code in JavaScript *(event path)*

**Type:** `n8n-nodes-base.code`

The Event Extractor returns the JSON inside a string field (`$json.output`). This node parses it into a proper JSON object so the Google Calendar node can read `start`, `end`, `title`, and `description` as structured fields:

```js
return [{ json: JSON.parse($input.first().json.output) }];
```

---

### Create an event *(event path)*

**Type:** `n8n-nodes-base.googleCalendar` (operation: create)

Creates a Google Calendar event using the parsed fields:
- `start` → event start datetime
- `end` → event end datetime
- `summary` → event title (mapped from `title`)
- `description` → event description

Uses OAuth 2.0 credentials connected to the target Google account.

---

### Append row in sheet *(sheet path)*

**Type:** `n8n-nodes-base.googleSheets` (operation: append)

Appends a new row to a Google Sheets document with four columns:

| Column | Source |
|---|---|
| `received_at` | `$('Edit Fields').item.json.received_at` |
| `from` | `$('Edit Fields').item.json.from` |
| `summary` | `$('AI summary').item.json.output` |
| `type` | `$('AI Classifier').item.json.output` |

This creates a continuously growing log of processed emails.

---

### Remove label from message

**Type:** `n8n-nodes-base.gmail` (operation: removeLabels)

Removes the `UNREAD` label from the Gmail message using its `email_id`. This marks the email as read so the `is:unread` filter in the first node will never pick it up again.

Configured with `onError: continueRegularOutput` — if the label removal fails (e.g. already read), the loop still continues.

---

## Groq LLM Nodes

Three separate Groq language model nodes are wired as sub-node dependencies:

| Node name | Model | Connected to |
|---|---|---|
| Groq Chat Model | `openai/gpt-oss-20b` | AI Summary |
| Groq Chat Model2 | `openai/gpt-oss-20b` | AI Classifier |
| Groq Chat Model1 | `openai/gpt-oss-120b` | Event Extractor |

The Event Extractor uses the larger `120b` model because structured JSON extraction from free-form text benefits from higher reasoning capacity.

---

## Data Flow Example

**Input email:** "Your bootcamp batch starts May 25 at 9:00 AM"

```
Edit Fields      → email_content built from subject + snippet + sender
AI Summary       → "Exlearn Technologies announces the Data Analytics bootcamp batch starting May 25, 2026 at 9:00 AM. No action required."
AI Classifier    → "event"
Switch           → routes to event path
Event Extractor  → { "title": "...", "start": "2026-05-25T09:00:00+01:00", ... }
Code             → parses JSON string to object
Create event     → Google Calendar event created
Remove label     → email marked as read
```

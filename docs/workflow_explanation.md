# Workflow Explanation

A detailed walk-through of the processing logic, AI prompts, and design decisions behind the email automation agent.

---

## The Two-Path Design

Every email enters a single pipeline and exits through one of two paths:

```
email → summarize → classify → route
                                ├── event  → extract datetime → Google Calendar
                                └── sheet  → log summary     → Google Sheets
```

The separation is intentional: actionable emails with a date and time become calendar events automatically; everything else is stored as a searchable log. The Gmail `is:unread` filter combined with the mark-as-read step at the end ensures strict at-most-once processing — no email is ever handled twice.

---

## Phase 1: Ingestion

### Schedule Trigger → Get many messages

The workflow wakes up every minute. The Gmail node queries `is:unread` and returns up to 10 messages. The limit of 10 keeps execution time predictable and avoids Google API rate limits. If there are more than 10 unread emails, the next trigger cycle picks up the remainder.

The Gmail node returns raw message objects with all metadata already simplified by n8n's built-in Gmail integration: sender, recipient, subject, snippet, label names, and the `internalDate` Unix timestamp in milliseconds.

### Loop Over Items

The loop node wraps the entire per-email pipeline. It feeds one email at a time into the next node and waits for the full processing chain to complete before fetching the next item. When all emails are processed, the loop naturally exits and the execution ends.

---

## Phase 2: Field Preparation

### Edit Fields

The raw Gmail payload has nested structures and raw timestamps. This node flattens everything into a single JSON object with consistent field names:

- `email_id` — used to mark the email as read at the end
- `received_at` — ISO string converted from Unix milliseconds
- `labels` — array of label objects flattened to a comma-separated string
- `email_content` — a multi-line text block that concatenates From, Subject, Snippet, and Labels into a format suitable for an LLM prompt

The `email_content` field is what every AI node reads. It is built with escaped quotes to handle edge cases in sender names and subject lines.

---

## Phase 3: AI Processing

### AI Summary

The summarizer sees the structured email fields (from, subject, snippet, labels) and returns 2–3 plain sentences. The prompt explicitly forbids preamble, labels, markdown, or any framing language — only the summary text itself.

The summary serves two purposes:
1. It is stored in Google Sheets so the log is human-readable without opening emails
2. It is passed to the AI Classifier as the text to classify, which is more efficient than re-reading the full email

**System prompt extract:**
```
You are an email summarizer.
```

**User prompt logic:**
```
Read the email and return ONLY a concise 2-3 sentence summary.
Focus on: what the email is about, who sent it, what action (if any) is needed.
No preamble. No labels. Plain text only.
```

### AI Classifier

The classifier reads the summary output and returns a single word: `event` or `sheet`. The prompt is written defensively — it lists explicit examples of what counts as an event (meetings, interviews, appointments, calls, schedules, calendar invitations, reminders with date/time) and what falls into sheet (newsletters, job alerts, promotions, notifications, invoices, updates).

The dual-prompt design (system prompt defines the role; user prompt provides the data and repeats the rules) reinforces the single-word output constraint. The Groq models occasionally add punctuation or trailing whitespace — the switch node uses strict equality so the classifier prompt must be reliable.

**Why classify the summary instead of the raw email?**  
The summary is shorter and already filtered to the key facts. It gives the classifier a cleaner signal and reduces token usage.

---

## Phase 4: Routing

### Switch

The switch node has two rules:
- Output 0: `$json.output === "event"` → event path
- Output 1: `$json.output === "sheet"` → sheet path

All items that do not match either rule are dropped (no third output). In practice this only happens if the classifier hallucinates something other than `event` or `sheet`.

---

## Phase 5a: Event Path

### Event Extractor

This node receives `email_content` (the full structured text from Edit Fields) and returns a JSON object with exactly four fields:

```json
{
  "title": "Short event title",
  "start": "2026-05-25T09:00:00+01:00",
  "end":   "2026-05-25T10:00:00+01:00",
  "description": "Human-readable event description"
}
```

The prompt enforces:
- ISO 8601 format with Morocco timezone offset (`+01:00`)
- Default end = start + 1 hour when no end time is stated in the email
- Raw JSON only — no markdown fences, no explanation

The larger `gpt-oss-120b` model is used here because structured extraction from free-form text has higher failure modes than simple classification.

### Code in JavaScript

The Event Extractor node wraps its output in `$json.output` as a string. The Google Calendar node expects structured fields (`start`, `end`), not a raw string. This JavaScript node does the minimal conversion:

```js
return [{ json: JSON.parse($input.first().json.output) }];
```

No validation is done here — if the LLM returns malformed JSON, the node will throw and the loop will skip to the next email.

### Create an event

The Google Calendar node receives `start`, `end`, and `description` from the parsed JSON. The `title` field is mapped to the Calendar API's `summary` field. The event is created in the calendar configured in the node settings.

---

## Phase 5b: Sheet Path

### Append row in sheet

Four columns are written to the spreadsheet:

| Column | Value | Source node |
|---|---|---|
| `received_at` | ISO timestamp of when the email arrived | Edit Fields |
| `from` | Full sender string including display name | Edit Fields |
| `summary` | 2–3 sentence AI summary | AI Summary |
| `type` | Always `sheet` on this path | AI Classifier |

The `type` column is included for future use — if you add a third category (e.g. `invoice`), the sheet log will distinguish it without schema changes.

---

## Phase 6: Cleanup

### Remove label from message

Regardless of which path was taken, both paths converge at this node. It calls the Gmail API to remove the `UNREAD` label from the processed message using its `email_id`.

This is the "ack" step — without it, the same email would be fetched again on the next trigger cycle. The node is set to `onError: continueRegularOutput` so a transient Gmail API error does not halt the loop. In the worst case an email gets processed twice on the next run, but that is rare.

After this node, the item is fed back into the **Loop Over Items** node's input, which fetches the next email from the batch.

---

## Design Decisions

**Why Groq instead of OpenAI directly?**  
Groq provides very low-latency inference which matters when processing 10 emails sequentially per minute. The API is OpenAI-compatible so the n8n Groq nodes work identically to the OpenAI nodes.

**Why not use Gmail triggers instead of a schedule?**  
Gmail push notifications via PubSub require a publicly accessible webhook URL. A scheduled poll works reliably on a local or self-hosted n8n instance without any infrastructure changes.

**Why is the summary generated before classification?**  
The summary is the primary artifact written to Google Sheets. Generating it first means the classifier can work on a shorter, cleaner input, and there is no repeated AI call for the sheet path.

**Why batch size 10?**  
Small enough that a single workflow execution finishes in under 30 seconds on a standard Groq free tier. Large enough to clear a typical inbox burst quickly.

---

## Example Run

**Incoming email:**
```
From: Exlearn Technologies <info@exlearn.ma>
Subject: Your bootcamp batch starts May 25 — welcome!
Snippet: The Data Analytics & Science with AI Bootcamp batch begins on 25 May 2026 at 9:00 AM.
```

**AI Summary output:**
```
Exlearn Technologies announces that the Data Analytics & Science with AI Bootcamp batch begins on 25 May 2026 at 9:00 AM. The email welcomes the recipient to the program. No immediate action is required.
```

**AI Classifier output:**
```
event
```

**Event Extractor output:**
```json
{
  "title": "Data Analytics & Science with AI Bootcamp start",
  "start": "2026-05-25T09:00:00+01:00",
  "end":   "2026-05-25T10:00:00+01:00",
  "description": "The Data Analytics & Science with AI Bootcamp batch begins on 25 May 2026, as announced by Exlearn Technologies."
}
```

**Result:** A Google Calendar event is created at 09:00 Morocco time on 25 May 2026. The email is marked as read.

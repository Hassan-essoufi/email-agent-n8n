# Setup Guide

This guide walks through everything needed to run the email automation agent from scratch.

---

## Prerequisites

- [Docker](https://www.docker.com/get-started) and Docker Compose installed
- A **Google account** with Gmail, Google Sheets, and Google Calendar enabled
- A **Groq API key** — free tier at [console.groq.com](https://console.groq.com)
- A **Google Cloud project** with OAuth credentials (for Gmail, Sheets, Calendar)

---

## Step 1 — Start n8n with Docker

From the project root, run:

```bash
docker compose up -d
```

The `docker-compose.yml` starts an n8n instance with a persistent volume. Once running, open:

```
http://localhost:5678
```

Create your n8n account on first launch. This account is local and not synced to the cloud.

---

## Step 2 — Set Up Google OAuth Credentials

You need a single **Google Cloud OAuth 2.0 Client** that covers Gmail, Sheets, and Calendar.

### 2a. Create a Google Cloud project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (e.g. `email-agent-n8n`)
3. Enable the following APIs under **APIs & Services → Library**:
   - Gmail API
   - Google Calendar API
   - Google Sheets API

### 2b. Create OAuth credentials

1. Go to **APIs & Services → Credentials**
2. Click **Create Credentials → OAuth client ID**
3. Application type: **Web application**
4. Add the n8n redirect URI to **Authorized redirect URIs**:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
5. Save and note your **Client ID** and **Client Secret**

### 2c. Configure the OAuth consent screen

1. Go to **APIs & Services → OAuth consent screen**
2. Set user type to **External**
3. Add your Gmail address as a **Test user**
4. Add the required scopes:
   - `https://www.googleapis.com/auth/gmail.modify`
   - `https://www.googleapis.com/auth/calendar`
   - `https://www.googleapis.com/auth/spreadsheets`

---

## Step 3 — Add Credentials in n8n

Open n8n at `http://localhost:5678` and navigate to **Credentials → New**.

### Gmail

- Select **Gmail OAuth2 API**
- Enter your Google Cloud **Client ID** and **Client Secret**
- Click **Connect** and authorize with your Google account

### Google Calendar

- Select **Google Calendar OAuth2 API**
- Use the same Client ID and Client Secret
- Authorize again

### Google Sheets

- Select **Google Sheets OAuth2 API**
- Use the same Client ID and Client Secret
- Authorize again

### Groq

- Select **Groq** (appears after installing the Groq node — it is bundled with n8n by default)
- Paste your **Groq API key**
- Save

---

## Step 4 — Import the Workflow

1. In n8n, go to **Workflows**
2. Click **⋮ → Import from file**
3. Select `workflows/email_automation_agent.json`

The workflow will open in the editor with all nodes visible.

---

## Step 5 — Configure Your Targets

Two nodes need to point to your own Google resources.

### Google Sheets — Append row in sheet

1. Open the **Append row in sheet** node
2. Under **Document**, click the dropdown and select your target spreadsheet (or paste its ID from the URL)
3. Under **Sheet**, select the target sheet tab
4. Make sure the sheet has these column headers in row 1:
   ```
   received_at | from | summary | type
   ```

### Google Calendar — Create an event

1. Open the **Create an event** node
2. Under **Calendar**, select your calendar (usually your Gmail address)
3. Save and close

---

## Step 6 — Verify Node Credentials

After importing, each node that uses a credential will show an orange warning if the credential ID from the original export does not match your local credentials. Fix each one:

1. Click the node
2. Under **Credential**, open the dropdown
3. Select the credential you created in Step 3
4. Save

Nodes that need credentials: **Get many messages**, **Append row in sheet**, **Create an event**, **Remove label from message**, and all three **Groq Chat Model** nodes.

---

## Step 7 — Test Before Activating

1. Make sure at least one email in your inbox is unread
2. Click **Execute workflow** (the play button at the top of the canvas)
3. Watch each node turn green as it completes
4. Verify:
   - A row appeared in your Google Sheet (for non-event emails)
   - A calendar event was created (for event emails)
   - The processed emails are now marked as read

---

## Step 8 — Activate the Workflow

Toggle the **Active** switch at the top right of the workflow editor. The workflow will now run automatically every minute.

---

## Adjusting the Schedule

By default the workflow triggers every **1 minute**. To change it:

1. Open the **Schedule Trigger** node
2. Change the interval value and unit
3. Save

Common alternatives:
- Every 5 minutes: set field to `minutes`, value to `5`
- Every hour: set field to `hours`, value to `1`

---

## Environment Variables (Optional)

If you want to pass the Groq API key or other secrets via environment variable instead of the n8n credential store, add them to the `docker-compose.yml` under the `environment` key and reference them in n8n using the `$env` variable.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Gmail node returns 0 emails | Make sure `is:unread` filter is correct and there are unread emails in the inbox |
| OAuth error on Gmail | Re-authorize the Gmail credential in n8n |
| Event Extractor returns malformed JSON | Check the Groq API key and model name in Groq Chat Model1 |
| Google Calendar event not created | Confirm the calendar is selected and the OAuth token has Calendar scope |
| Emails not marked as read | The Remove label node has `onError: continueRegularOutput` — check Gmail API permissions |
| n8n not accessible | Run `docker compose logs n8n` to check for startup errors |

# N8N Workflows — NHS Westshire ICB FOI Agent

This directory contains two N8N workflow files that power the FOI Agent automation. Both are importable directly into any N8N v1.x instance.

---

## Workflows

### 1. `foi-email-workflow.json` — Email-Based FOI Processing

**Trigger:** Scheduled (every 2 minutes, checks for unread email)  
**Purpose:** Polls the FOI inbox, processes incoming emails, classifies the request type, generates a draft response, sends an acknowledgement, and alerts the FOI Lead.

**Full flow:**
```
Schedule (2 min) → Read IMAP (FOI inbox) → IF: new emails?
  → Extract email + generate case ref
  → Claude API: Classify (FOI/EIR/SAR/GENERAL)
  → Switch: route by type
  → Claude API: Disclosure analysis + draft response
  → Send acknowledgement to requester
  → Send internal alert to FOI Lead
  → Log to Google Sheets (FOI Case Register)
```

### 2. `foi-form-workflow.json` — Web Form FOI Processing

**Trigger:** N8N Form Trigger (webhook, real-time)  
**Purpose:** Receives submissions from the public FOI request form on the ICB website, processes them through the same Claude-powered pipeline, and logs them to the same Google Sheets case register.

**Full flow:**
```
Form Trigger (webhook) → Format as FOI Request + generate case ref
  → Claude API: Classify + Analyse + Draft response
  → Send confirmation to requester
  → Send internal alert to FOI Lead
  → Log to Google Sheets (FOI Case Register)
```

---

## How to Import

1. Open your N8N instance (default: `http://localhost:5678`)
2. Navigate to **Workflows** in the left sidebar
3. Click the **+** button and select **Import from file**
4. Select the `.json` file from this directory
5. Click **Import**
6. Configure credentials (see below) before activating

---

## Required Credentials

You must create these credentials in N8N **before** activating the workflows:

| Credential Name | Type | Purpose |
|---|---|---|
| `FOI IMAP Account` | IMAP | Reads unread email from foi@westshire.icb.nhs.uk |
| `NHS.net SMTP` | SMTP | Sends acknowledgements and internal alerts |
| `Anthropic API Key` | HTTP Header Auth | Calls the Claude API (header name: `x-api-key`) |
| `Google Sheets` | Google Sheets OAuth2 | Logs cases to the FOI Case Register |

See `/guides/01-imap-email-setup.md`, `/guides/02-n8n-setup.md`, and `/guides/03-anthropic-api-setup.md` for step-by-step setup instructions.

---

## Environment Variables

The workflows reference these environment variables, which must be set in N8N's environment before activation:

| Variable | Example Value | Required By |
|---|---|---|
| `ANTHROPIC_API_KEY` | `sk-ant-...` | Both workflows |
| `FOI_EMAIL_ADDRESS` | `foi@westshire.icb.nhs.uk` | Both workflows |
| `FOI_REVIEWER_EMAIL` | `diane.okafor@westshire.icb.nhs.uk` | Both workflows |
| `GOOGLE_SHEETS_ID` | `1BxiMVs0XRA5nFMd...` | Both workflows |

Set environment variables either in your `.env` file (for Docker) or in N8N's Settings → Environment.

---

## Google Sheets: FOI Case Register

Both workflows log to a sheet named **"FOI Case Register"** within the Google Sheets document identified by `GOOGLE_SHEETS_ID`. Create this sheet with the following column headers in row 1:

```
Case Reference | Date Received | Requester Name | Requester Email | Subject | 
Request Type | Subject Matter | Deadline | Draft Decision | Exemptions | 
Escalation Required | Escalation Reason | Status | Logged At | Acknowledgement Sent
```

---

## Workflow Architecture Notes

**Dual Claude API call (email workflow only):**  
The email workflow makes two calls to the Anthropic API per request:
1. **Classify** (max 1,000 tokens) — fast, lightweight classification of request type
2. **Analyse** (max 4,000 tokens) — full disclosure analysis with knowledge base, draft response, exemption assessment

This pattern keeps the first call fast (for routing decisions) while allowing the second call to be more thorough.

**Prompt caching:**  
Both workflows use `anthropic-beta: prompt-caching-2024-07-31` with `cache_control: {"type": "ephemeral"}` on the system prompt. For high-volume deployments, this significantly reduces costs — the large knowledge base system prompt is cached after the first call.

**Error handling:**  
Both workflows include fallback JSON structures if the Claude API response cannot be parsed. In fallback mode, the case is flagged for manual review and an escalation alert is sent to the FOI Lead.

---

## Activating the Workflows

Once credentials and environment variables are configured:

1. Open the workflow in N8N
2. Click the toggle at the top right to set it to **Active**
3. The email workflow will begin polling on its 2-minute schedule
4. The form workflow will activate its webhook URL immediately

To find the webhook URL for the form workflow: open the Form Trigger node and copy the Production URL. This is the URL to embed in the ICB website's FOI request form.

# Guide 04: Testing Guide

## Overview

This guide explains how to test the FOI Agent workflows before going live. Testing involves sending realistic FOI request emails to a test inbox, verifying the agent's classification and draft responses, and checking the case register. Run through the full regression checklist at the end before activating on the live FOI inbox.

---

## 1. Test Environment Setup

**Do not test on the live FOI inbox.** Set up a separate test environment:

### Step 1: Create a test email address

Create a separate email address for testing, e.g.:
- `foi-test@westshire.icb.nhs.uk` (if your email admin can create a test mailbox)
- A personal Gmail account used only for testing
- A Mailinator or temporary email address (for send-only testing)

### Step 2: Configure a test N8N instance

Run a second N8N instance on a different port for testing:

```bash
# Create a test directory
mkdir ~/n8n-foi-test
cd ~/n8n-foi-test

# Copy and modify docker-compose.yml — change port 5678 to 5679
# In .env file, set FOI_EMAIL_ADDRESS to your test address
# Set FOI_REVIEWER_EMAIL to your own email for testing
```

Alternatively, in your main N8N instance, create copies of the workflows with `(TEST)` in the name and configure them to use test credentials.

### Step 3: Set up a test Google Sheet

Create a separate Google Sheet named `FOI Case Register (TEST)` and configure the test workflows to log to this sheet. This prevents test data from polluting the live case register.

### Step 4: Use a test Anthropic API key

Consider creating a separate API key named `FOI Agent Testing` in the Anthropic Console with a low spending limit (e.g. £5/month) to prevent runaway costs during testing.

---

## 2. Sample Test Requests

Use the sample requests from `examples/sample-foi-requests.md`. For automated testing, send these as actual emails to your test inbox.

**Quick-start test set (5 requests covering key scenarios):**

1. **Simple disclosure (from the disclosure log):**  
   Send Request #1 from the sample requests (staff headcount by AfC band). Expected: `DISCLOSE`, should reference previous response FOI-2024-0001.

2. **s.21 referral (published information):**  
   Send Request #5 (Board meeting minutes). Expected: `REFER` under s.21.

3. **Partial disclosure with s.40:**  
   Send Request #9 (named individual salary). Expected: `PARTIAL`, s.40 exemption applied, escalation to DPO.

4. **EIR request:**  
   Send Request #22 (environmental information). Expected: reclassification as EIR, not FOI.

5. **Escalation trigger:**  
   Send Request #18 (journalist request). Expected: `escalation_required: true`, escalation reason includes journalist/media.

---

## 3. Expected Outputs

### What a good acknowledgement looks like

A well-formed acknowledgement should contain all of:
- The case reference (e.g. `FOI-2026-1042`) prominently displayed
- The requester's name (or "Dear [name]" greeting)
- Confirmation that the request has been received under the correct legislation (FOIA 2000 or EIR 2004)
- The exact statutory deadline (calculated date, not just "20 working days")
- Contact details and sign-off (`Freedom of Information Team, NHS Westshire ICB`)

**Example of a correct acknowledgement:**
```
Dear Ms Williams,

Thank you for your Freedom of Information request received on Wednesday, 14 May 2026.

Your request has been logged with case reference FOI-2026-1042.

We will respond to your request by Wednesday, 10 June 2026, as required by Section 10 of the Freedom of Information Act 2000.

If you have any queries, please contact us at foi@westshire.icb.nhs.uk quoting your case reference.

Freedom of Information Team
NHS Westshire Integrated Care Board
Westshire House, 12 Cathedral Quarter, Westminster-upon-Avon WS1 2AB
foi@westshire.icb.nhs.uk | www.westshireicb.nhs.uk
```

**Failure indicators:** Missing case reference; incorrect legislation cited; deadline calculated incorrectly (e.g. exactly 20 calendar days rather than 20 working days); no sign-off.

### What a good draft response looks like

A well-formed draft response should:
- State clearly whether the information is held
- If disclosing: provide the information clearly or state where it can be found
- If refusing: cite the exact FOIA section (e.g. "Section 40(2)"), explain what the exemption covers, and (for qualified exemptions) include a public interest argument
- Include the requester's right to request an internal review
- Include the right to complain to the ICO
- Be addressed to the requester (not a generic template)
- Be signed appropriately

---

## 4. Manual Workflow Execution

You can trigger workflows manually in N8N without waiting for the schedule:

### For the email workflow:

1. Open the `NHS Westshire ICB — FOI Email Processing` workflow
2. Click **Test Workflow** (the play button at the bottom left of the canvas)
3. N8N will execute from the Schedule Trigger node
4. Watch the execution progress — each node will show a green tick or red X as it runs
5. Click any node to inspect its input and output data

**To test with a specific email without waiting:**
1. Open the **Code: Extract Email** node
2. Click **Execute Node** while providing a mock input:
```json
{
  "from": "\"Jane Williams\" <jane.williams@example.com>",
  "subject": "FOI Request - staff numbers",
  "text": "Dear Sir/Madam, I would like to request the total number of staff employed by NHS Westshire ICB, broken down by Agenda for Change pay band. Regards, Jane Williams",
  "date": "2026-05-14T10:00:00.000Z"
}
```

### For the form workflow:

1. Open the `NHS Westshire ICB — FOI Web Form Processing` workflow
2. Copy the **test webhook URL** from the Form Trigger node
3. Either:
   - Open the URL in a browser and fill in the form, or
   - Send a POST request with curl:
```bash
curl -X POST https://your-n8n-instance.url/webhook-test/foi-form-webhook-001 \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "James Okonkwo",
    "email_address": "j.okonkwo@example.com",
    "request_details": "Please provide the total value of contracts held with Westshire University Hospitals NHS Trust.",
    "preferred_format": "Electronic (email/PDF)"
  }'
```

---

## 5. Inspection — Viewing Execution Logs

N8N stores full execution logs for every workflow run.

1. In N8N, click **Executions** in the left sidebar
2. You will see a list of recent workflow executions, with timestamps and status
3. Click any execution to open the detailed view
4. Click any node in the execution to see:
   - **Input data:** what the node received
   - **Output data:** what the node produced
   - **Error details** (if the node failed)

**Useful things to check:**
- Code: Extract Email → verify case reference and deadline calculation
- HTTP Request nodes → verify the API call was made correctly; check the response status code
- Code: Parse Classification → verify the JSON was parsed correctly
- Send Acknowledgement → verify the email was sent (check "status" in the output)
- Google Sheets → verify the row was appended

**Log retention:** By default, N8N keeps execution logs for 7 days (168 hours). Adjust `EXECUTIONS_DATA_MAX_AGE` in your environment to change this.

---

## 6. Common Failure Modes

| Failure | Symptom | Diagnosis | Fix |
|---|---|---|---|
| API key invalid | HTTP 401 on Claude API nodes | Check Anthropic API Key credential | Regenerate key in Anthropic Console; update N8N credential |
| IMAP auth failed | Read IMAP node fails with authentication error | Check IMAP credential | Verify app password; regenerate if expired |
| No emails found | Workflow runs but IF node stops | Normal when inbox is empty | Send a test email; check it is arriving at the test inbox |
| Malformed JSON from Claude | Code: Parse Classification fails or returns fallback | Claude returned non-JSON text | Check the system prompt; Claude usually returns valid JSON but rarely wraps it in markdown — fallback handling should catch this |
| SMTP relay error | Send Acknowledgement fails | Check SMTP credential or port | Verify SMTP host, port (587 for STARTTLS), and app password |
| Google Sheets: permission denied | Log Case node fails | OAuth scope or sheet permissions | Re-authorise Google Sheets credential; ensure the sheet ID is correct |
| Deadline calculated wrong | Wrong date in acknowledgement | Bank holiday or timezone issue | Check `GENERIC_TIMEZONE=Europe/London` is set; verify bank holiday list in Code node |
| Duplicate processing | Same email processed twice | `markSeen` not working | Ensure `markSeen: true` in Read IMAP node; check IMAP server supports \Seen flag |

---

## 7. Regression Test Checklist

Run through this checklist before activating the workflows on the live FOI inbox:

- [ ] **1. Schedule trigger fires correctly** — manually trigger the email workflow; confirm it runs without errors on an empty inbox
- [ ] **2. IMAP connection reads new emails** — send a test email to the FOI test inbox; confirm the workflow picks it up within 2 minutes
- [ ] **3. Case reference is generated** — check that the case reference follows the format `FOI-YYYY-NNNN` (4-digit year, 4-digit number)
- [ ] **4. 20-working-day deadline is correctly calculated** — send a test request on a Monday; verify the deadline is the Monday 4 weeks later (skipping weekends), not exactly 28 calendar days later
- [ ] **5. Classification is correct** — test with at least one FOI request, one EIR request, one SAR, and one general query; verify each is classified correctly
- [ ] **6. Escalation flags correctly** — send a test request from a fictional journalist name in the body (e.g. "I am a reporter for the Westshire Gazette"); verify `escalation_required: true` in the classification
- [ ] **7. Acknowledgement email is sent to the requester** — verify the acknowledgement arrives at the test requester address, contains the correct case reference and deadline, and is correctly signed off
- [ ] **8. Internal alert is sent to the reviewer** — verify the internal alert arrives at the test reviewer address, contains the draft response and all key case details
- [ ] **9. Case is logged to Google Sheets** — verify a new row appears in the test FOI Case Register sheet with all fields populated correctly
- [ ] **10. Form workflow works end-to-end** — submit a test request via the web form; verify all of the above happen correctly for form submissions

**Sign-off:** Before going live, have the FOI Lead review at least 3 test case outputs (acknowledgement + draft response) and confirm they are of acceptable quality for human review.

---

## 8. Performance Baseline

After go-live, establish a performance baseline over the first month:

- Average end-to-end processing time (email received to acknowledgement sent): target under 5 minutes
- Classification accuracy: ask the FOI Lead to rate each Claude classification as correct/incorrect for the first month
- Draft response quality: ask the FOI Lead to rate each draft as "good", "needs minor edit", or "needs major rework"
- API costs: review monthly Anthropic billing; should be under £5/month at NHS ICB volumes

Use these baselines to tune the system prompt and knowledge base over time.

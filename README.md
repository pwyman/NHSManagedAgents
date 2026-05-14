# NHS Managed Agents — FOI Agent Example

> **NOT FOR PRODUCTION USE** — This is a reference implementation and learning example. Before deploying in an NHS environment, review all compliance requirements with your IG team, DPO, and clinical governance leads.

A complete, working example of an AI-powered Freedom of Information agent for an NHS Integrated Care Board, built with Claude (Anthropic) and N8N. This is the third in the NHS Managed Agents series, demonstrating how AI agents can automate routine administrative work in NHS organisations while keeping humans in control of every consequential decision.

---

## What This Repository Contains

```
NHSManagedAgents/
├── README.md                          ← You are here
├── .env.example                       ← Environment variable template
│
├── agents/
│   └── foi-agent.md                   ← Agent definition (10-section structure)
│
├── knowledge-base/
│   ├── icb-profile.md                 ← Complete ICB factual reference
│   ├── foi-disclosure-log.md          ← 25 previous FOI responses (Q&A format)
│   └── foi-exemptions-reference.md    ← FOIA 2000 exemptions guide for the agent
│
├── n8n-workflows/
│   ├── README.md                      ← Workflow import and setup guide
│   ├── foi-email-workflow.json        ← Email-based FOI processing (importable)
│   └── foi-form-workflow.json         ← Web form FOI processing (importable)
│
├── guides/
│   ├── 01-imap-email-setup.md         ← Gmail, M365, and NHS.net IMAP configuration
│   ├── 02-n8n-setup.md                ← N8N installation and deployment guide
│   ├── 03-anthropic-api-setup.md      ← Anthropic API key, costs, and caching
│   └── 04-testing-guide.md            ← Testing procedures and regression checklist
│
└── examples/
    └── sample-foi-requests.md         ← 25 realistic sample FOI requests for testing
```

---

## Quick Start (5 Steps)

1. **Clone this repository** and copy `.env.example` to `.env`
2. **Install N8N** using Docker (`guides/02-n8n-setup.md`)
3. **Configure credentials** — IMAP, SMTP, Anthropic API key, Google Sheets (`guides/01-imap-email-setup.md`, `guides/03-anthropic-api-setup.md`)
4. **Import workflows** — import both JSON files from `n8n-workflows/` into N8N
5. **Test** using the sample requests in `examples/sample-foi-requests.md`, then activate

**Prerequisites:** Docker, an Anthropic API key, and access to the FOI email inbox.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  INBOUND CHANNELS                                               │
│                                                                 │
│  ┌─────────────────┐        ┌─────────────────────────┐        │
│  │  foi@westshire  │        │  ICB Website            │        │
│  │  .icb.nhs.uk    │        │  FOI Request Form       │        │
│  │  (email inbox)  │        │  (web form / webhook)   │        │
│  └────────┬────────┘        └────────────┬────────────┘        │
│           │                              │                      │
└───────────┼──────────────────────────────┼──────────────────────┘
            │                              │
            ▼                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  N8N WORKFLOW ENGINE  (self-hosted; NHS infrastructure)         │
│                                                                 │
│  ┌─────────────┐  ┌───────────────┐  ┌────────────────────┐   │
│  │  Schedule   │  │  Extract &    │  │  Build Anthropic   │   │
│  │  Trigger    │→ │  Parse Email  │→ │  API Payload       │   │
│  │  (2 min)    │  │  + Case Ref   │  │  (w/ KB context)  │   │
│  └─────────────┘  └───────────────┘  └─────────┬──────────┘   │
│                                                  │              │
└──────────────────────────────────────────────────┼──────────────┘
                                                   │
                            ┌──────────────────────┼────────────────────────┐
                            │  DATA SOVEREIGNTY    │  BOUNDARY              │
                            │  OFFICIAL data only  │  (review before        │
                            │  Anthropic DPA       │  sending OFFICIAL-     │
                            │  in place            │  SENSITIVE)            │
                            └──────────────────────┼────────────────────────┘
                                                   │
                                                   ▼
                            ┌─────────────────────────────────────────┐
                            │  ANTHROPIC CLAUDE API                   │
                            │  (claude-sonnet-4-6)                    │
                            │                                         │
                            │  Call 1: Classify                       │
                            │  (FOI / EIR / SAR / GENERAL)           │
                            │                                         │
                            │  Call 2: Disclose Analysis              │
                            │  + Draft acknowledgement                │
                            │  + Draft response                       │
                            │  + Exemptions assessment                │
                            └───────────────────┬─────────────────────┘
                                                │
                                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  N8N (continued)                                                │
│                                                                 │
│  ┌──────────────────┐     ┌───────────────────────────────┐   │
│  │ Send ACK         │     │  Send Internal Alert          │   │
│  │ to Requester     │     │  to FOI Lead (Diane Okafor)   │   │
│  │ (automated)      │     │  with full draft for review   │   │
│  └──────────────────┘     └───────────────────────────────┘   │
│                                         │                       │
│  ┌──────────────────────────────────────▼────────────────────┐ │
│  │  Google Sheets — FOI Case Register (auto-logged)          │ │
│  └──────────────────────────────────────────────────────────-┘ │
└─────────────────────────────────────────────────────────────────┘
                                         │
                            ┌────────────▼────────────┐
                            │  HUMAN REVIEW           │
                            │  FOI Lead reviews draft  │
                            │  Approves or edits       │
                            │  Sends final response    │
                            │  (human decision always) │
                            └─────────────────────────┘
```

---

## File Descriptions

| File | Purpose | Used By |
|---|---|---|
| `agents/foi-agent.md` | The agent's identity, rules, autonomy boundaries, and escalation logic | Claude API system prompt |
| `knowledge-base/icb-profile.md` | Structured ICB facts for answering FOI queries | Claude API (included in analysis prompt) |
| `knowledge-base/foi-disclosure-log.md` | 25 previous FOI responses — helps identify duplicates and draft consistent responses | Claude API (included in analysis prompt) |
| `knowledge-base/foi-exemptions-reference.md` | FOIA 2000 and EIR exemptions reference guide | Claude API (included in analysis prompt) |
| `n8n-workflows/foi-email-workflow.json` | N8N workflow: polls email inbox, classifies, analyses, acknowledges | N8N import |
| `n8n-workflows/foi-form-workflow.json` | N8N workflow: processes web form submissions | N8N import |
| `.env.example` | Template for all required environment variables | Developer setup |
| `guides/01-imap-email-setup.md` | Step-by-step IMAP setup for Gmail, M365, and NHS.net | System administrator |
| `guides/02-n8n-setup.md` | N8N installation, Docker config, credential setup | System administrator |
| `guides/03-anthropic-api-setup.md` | API key creation, cost guidance, prompt caching | Developer |
| `guides/04-testing-guide.md` | Testing procedures and regression checklist | Developer / FOI Lead |
| `examples/sample-foi-requests.md` | 25 realistic FOI requests for testing | Developer / FOI Lead |

---

## Step-by-Step Deployment

### Step 1: Environment and infrastructure

```bash
# Clone the repository
git clone https://github.com/your-org/nhs-managed-agents-foi
cd nhs-managed-agents-foi

# Copy environment template
cp .env.example .env

# Edit .env with your values
nano .env
```

Minimum required values in `.env`:
- `ANTHROPIC_API_KEY` — from console.anthropic.com
- `FOI_EMAIL_ADDRESS` — the ICB's FOI inbox address
- `FOI_REVIEWER_EMAIL` — the FOI Lead's email
- `GOOGLE_SHEETS_ID` — your FOI Case Register sheet ID

### Step 2: Install and start N8N

```bash
# Using Docker (recommended)
docker-compose up -d

# Verify N8N is running
docker-compose logs -f | grep "Editor is now accessible"
```

See `guides/02-n8n-setup.md` for full installation instructions including HTTPS setup.

### Step 3: Configure credentials in N8N

Open N8N at `http://localhost:5678` (or your configured URL) and create:
1. `FOI IMAP Account` — IMAP credential for the FOI inbox
2. `NHS.net SMTP` — SMTP credential for sending emails
3. `Anthropic API Key` — HTTP Header Auth credential
4. `Google Sheets` — OAuth2 Google Sheets credential

See `guides/01-imap-email-setup.md` and `guides/03-anthropic-api-setup.md` for details.

### Step 4: Import workflows

In N8N:
1. Workflows → Import from file → select `n8n-workflows/foi-email-workflow.json`
2. Repeat for `n8n-workflows/foi-form-workflow.json`

### Step 5: Set up the FOI Case Register

Create a Google Sheet named `FOI Case Register` with these column headers:
```
Case Reference | Date Received | Requester Name | Requester Email | Subject |
Request Type | Subject Matter | Deadline | Draft Decision | Exemptions |
Escalation Required | Escalation Reason | Status | Logged At
```

Copy the Sheet ID (from the URL) into your `.env` and N8N environment.

### Step 6: Test

Send the first 5 requests from `examples/sample-foi-requests.md` to a test inbox. Verify classification, acknowledgement content, draft responses, and case register logging. See `guides/04-testing-guide.md` for the full test procedure.

### Step 7: Go live

Once testing passes:
1. Switch N8N credentials to point to the live FOI inbox
2. Activate both workflows in N8N
3. Brief the FOI Lead on the internal alert format and the review/approval process
4. Monitor the first 10 live cases closely

---

## How the FOI Agent Works

```
                    Request received
                           │
                           ▼
          ┌────────────────────────────────────┐
          │  Classification (Claude Call 1)     │
          │  FOI │ EIR │ SAR │ GENERAL │ UNCLEAR│
          └────────────────┬───────────────────┘
                           │
                           ▼
                  ┌────────────────┐
                  │  Escalation     │ ──── YES ──→ Alert FOI Lead
                  │  check         │             (journalist? MP?
                  └────────┬───────┘              vexatious pattern?
                           │ NO                   personal data?)
                           ▼
          ┌────────────────────────────────────┐
          │  Disclosure Analysis (Claude Call 2)│
          │  • Search knowledge base            │
          │  • Identify exemptions              │
          │  • Draft acknowledgement            │
          │  • Draft response letter            │
          └────────────────┬───────────────────┘
                           │
                           ▼
          ┌────────────────────────────────────┐
          │  Automated outputs (no human yet):  │
          │  • ACK email → requester            │
          │  • Alert email → FOI Lead           │
          │  • Row → Google Sheets              │
          └────────────────┬───────────────────┘
                           │
                           ▼
          ┌────────────────────────────────────┐
          │  HUMAN REVIEW (FOI Lead)            │
          │  Reviews draft; approves or edits   │
          │  Signs off the final response       │
          └────────────────┬───────────────────┘
                           │
                           ▼
                  Final response sent
                  to requester by
                  FOI Lead (or delegate)
```

---

## Knowledge Base Structure

The three knowledge base files are loaded into the Claude API prompt when analysing requests:

**`icb-profile.md`** — the "brain" of the agent: who the ICB is, what it does, key figures, financial facts, published policies, and staff data. Claude searches this to determine whether information is held and what the disclosed answer should be.

**`foi-disclosure-log.md`** — 25 previous FOI responses in structured format. Claude uses this to:
- Identify duplicate requests and refer to previous responses
- Ensure consistent answers to recurring questions
- Understand what information the ICB has previously considered disclosable

**`foi-exemptions-reference.md`** — a guide to FOIA 2000 exemptions and EIR exceptions. Claude uses this to identify applicable exemptions and draft public interest test arguments.

---

## Example Exchange

**Incoming email (14 May 2026, 10:14am):**
```
From: martin.price@westshire-citizen.co.uk
Subject: FOI Request - staff numbers and sickness

Dear FOI Officer,

I'd like to request, under the Freedom of Information Act:

1. How many people does NHS Westshire ICB employ?
2. What is the staff sickness absence rate?

Many thanks,
Martin Price
```

**Acknowledgement sent automatically (10:18am, 4 minutes later):**
```
Dear Mr Price,

Thank you for your Freedom of Information request received on Thursday, 14 May 2026.

Your request has been logged with case reference FOI-2026-4271.

We will respond to your request by Thursday, 11 June 2026, as required by Section 10 of the 
Freedom of Information Act 2000.

If you have any queries, please contact us at foi@westshire.icb.nhs.uk quoting your 
case reference FOI-2026-4271.

Freedom of Information Team
NHS Westshire Integrated Care Board
Westshire House, 12 Cathedral Quarter, Westminster-upon-Avon WS1 2AB
foi@westshire.icb.nhs.uk | www.westshireicb.nhs.uk
```

**Internal alert to FOI Lead (10:18am):**
```
[FOI CASE ALERT] FOI-2026-4271 — FOI: staff headcount and sickness absence — Deadline: 11 June 2026

Classification: FOI | DISCLOSE | Confidence: HIGH
Escalation: No

DRAFT RESPONSE (for your approval):

Dear Mr Price,

Thank you for your Freedom of Information request dated 14 May 2026, reference FOI-2026-4271.

NHS Westshire ICB employs 387 whole-time equivalent (WTE) staff as at the most recent 
available data point. 

The ICB's sickness absence rate is 4.2%, compared to the NHS national average of 5.7%.

Please note that similar information was previously disclosed in FOI-2024-0001 (staff 
headcount) and FOI-2024-0005 (sickness absence).

[Appeal rights text...]

Freedom of Information Team
NHS Westshire ICB
```

---

## NHS Compliance Notes

This example is designed with the following NHS compliance requirements in mind:

1. **FOIA 2000 compliance:** The agent handles the s.10 20-working-day deadline; generates s.17 refusal notices; includes appeal rights. All responses require human (FOI Lead) approval before sending.

2. **Information security:** The workflows operate on OFFICIAL data (FOI requests are not OFFICIAL-SENSITIVE). No patient data flows through this system. Review with your IG team before processing any request that contains personal data.

3. **Data sovereignty:** The Anthropic API processes data in the United States. The workflows use Anthropic's API under a Data Processing Agreement (DPA) that includes Standard Contractual Clauses for GDPR compliance. Assess this against your organisation's data residency requirements.

4. **Human oversight:** Every draft response requires FOI Lead approval. The agent never sends a final response to a requester. Escalation logic is hardcoded for journalists, MPs, personal data, and vexatious patterns.

5. **Audit trail:** Every case is logged to Google Sheets with timestamp, classification, and decision. N8N execution logs provide full audit of every API call and email action.

---

## The Broader NHS Managed Agents Architecture

This FOI Agent is one component of a larger 24-agent architecture for NHS ICBs. See the companion articles:

- **Article 1:** NHS Managed Agents — the strategic case (LinkedIn)
- **Article 2:** Under the Hood — the technical architecture
- **Article 3:** This repository — the FOI Agent working example

The full 24-agent network includes agents for complaints, commissioning contracts, finance support, PALS, digital systems, and more.

---

## Contributing

This is a public reference implementation. If you adapt this for your NHS organisation:

1. Replace all references to "NHS Westshire ICB" with your actual organisation
2. Update `knowledge-base/icb-profile.md` with your ICB's real data
3. Build your `knowledge-base/foi-disclosure-log.md` from your own disclosure log
4. Review the escalation rules in `agents/foi-agent.md` with your FOI Lead and DPO
5. Test thoroughly before going live (see `guides/04-testing-guide.md`)

Issues and pull requests welcome.

---

## Licence

This reference implementation is released under the MIT Licence. You are free to use, adapt, and redistribute it for any purpose, including NHS operational use. Attribution appreciated but not required.

The fictional "NHS Westshire ICB" is entirely invented for illustrative purposes. Any resemblance to real NHS organisations is coincidental.

---

*NHS Managed Agents · NHS Westshire ICB Example · Version 1.0 · May 2026*

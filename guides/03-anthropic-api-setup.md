# Guide 03: Anthropic API Setup

## Overview

The FOI Agent uses the Anthropic Claude API to classify incoming requests and draft responses. This guide covers creating your Anthropic account, generating an API key, configuring it in N8N, understanding costs, and optimising the setup with prompt caching.

---

## 1. Creating an Anthropic Account and API Key

### Step 1: Create an Anthropic account

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Click **Sign up** and create an account
3. Verify your email address
4. Add a payment method (credit/debit card or bank account)
   - For NHS organisations: use a purchase card or procurement card
   - Pre-paid credits are available — request via your procurement team if needed

### Step 2: Generate an API Key

1. In the Anthropic Console, go to **API Keys** (left sidebar)
2. Click **Create Key**
3. Name the key: `NHS Westshire ICB FOI Agent`
4. Optionally, set a monthly spending limit for the key (recommended — start with £50/month, review after 1 month)
5. Click **Create Key**
6. **Copy the key immediately** — it will not be shown again. It begins with `sk-ant-`.
7. Store the key securely (e.g. in a password manager, or directly in your `.env` file)

**Security:** API keys are powerful — anyone with the key can make API calls charged to your account. Never commit API keys to version control. Never paste them into emails or chat messages.

### Step 3: Add the API Key to your environment

Add the key to your `.env` file:

```bash
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
```

And to your N8N environment (see Guide 02, Section 6, Step 3).

---

## 2. Configuring the Credential in N8N

The FOI workflows use **HTTP Header Authentication** to send the API key to Anthropic.

1. In N8N, go to **Settings → Credentials → Add Credential**
2. Search for **HTTP Header Auth**
3. Name the credential exactly: `Anthropic API Key`
4. Fill in:
   - **Name:** `x-api-key`
   - **Value:** `sk-ant-your-actual-api-key`
5. Click **Save**

The HTTP Request nodes in the FOI workflows are already configured to use this credential name. No other changes are needed.

---

## 3. Understanding Costs

### Model Pricing (claude-sonnet-4-6)

The FOI workflows use `claude-sonnet-4-6`, Anthropic's production-grade model. Current pricing:

| Item | Price |
|---|---|
| Input tokens | $3.00 per million tokens |
| Output tokens | $15.00 per million tokens |
| Cache write tokens (prompt caching) | $3.75 per million tokens |
| Cache read tokens (prompt caching) | $0.30 per million tokens |

*Prices as of mid-2026. Check [anthropic.com/pricing](https://anthropic.com/pricing) for current rates.*

### Cost per FOI Request (Estimate)

Each FOI request triggers two Claude API calls:

**Call 1 — Classification:**
- System prompt: ~800 tokens (cached after first call)
- Input (email): ~300–600 tokens
- Output (JSON classification): ~200–400 tokens
- Estimated cost: £0.001–0.002 per request

**Call 2 — Disclosure Analysis + Draft Response:**
- System prompt: ~1,500 tokens (cached after first call)
- Input (email + knowledge base): ~1,500–2,500 tokens
- Output (draft acknowledgement + draft response): ~600–2,000 tokens
- Estimated cost: £0.003–0.010 per request

**Total estimated cost per FOI request: £0.004–0.012 (~0.4p to 1.2p)**

At NHS Westshire ICB's volume (approximately 87 FOI requests per year), the **annual API cost is approximately £0.35 – £1.05**. This is negligible. Even at 10x volume, costs remain well under £15/year.

For organisations with much higher FOI volumes (e.g. large NHS trusts receiving 500+ requests/year), annual costs would still be under £10.

### Cost monitoring

Set a monthly budget alert in the Anthropic Console:
1. Go to **Billing → Usage limits**
2. Set a monthly limit (e.g. £20 for a small ICB)
3. Set an alert at 80% of the limit

---

## 4. Rate Limits and Error Handling

### Anthropic rate limits

Default rate limits for the `claude-sonnet-4-6` model (Tier 1 API access):

| Limit | Value |
|---|---|
| Requests per minute (RPM) | 50 |
| Tokens per minute (TPM) | 40,000 |
| Tokens per day | 1,000,000 |

For NHS ICB FOI volumes, you will not come close to these limits. Even if all 87 annual requests arrived simultaneously, that would be well within the per-minute limits.

### Error handling in the workflows

The FOI workflows include fallback handling for API errors:

- **HTTP 401 (Invalid API key):** The workflow will fail with "Authentication failed". Check your `Anthropic API Key` credential and ensure the key has not been revoked.
- **HTTP 429 (Rate limit exceeded):** Unlikely at NHS ICB volumes. If it occurs, the workflow will fail on that execution. The next scheduled run (2 minutes later) will process the email again.
- **HTTP 500 (Anthropic server error):** Temporary. The workflow will retry on the next scheduled run.
- **Malformed JSON response:** The Code nodes include fallback handling — if Claude's response cannot be parsed as JSON, the case is flagged for manual review and an alert is sent to the FOI Lead.

### Setting up workflow error notifications

1. In N8N, go to **Settings → Workflows**
2. Under **Error Workflow**, select or create an error-handling workflow that emails the system administrator
3. Alternatively, use N8N's built-in error email setting:
   - **Settings → Users** → set the admin email address for error notifications

---

## 5. Choosing the Right Model

The FOI workflows are configured to use `claude-sonnet-4-6`. Here is guidance on model selection:

| Model | Use Case | Cost |
|---|---|---|
| `claude-sonnet-4-6` | **Production** — high-quality responses, good balance of speed and quality | Standard |
| `claude-haiku-4-5` | **Testing / high volume** — faster, cheaper, lower quality | ~8x cheaper than Sonnet |
| `claude-opus-4-5` | **Complex edge cases** — highest quality, best reasoning | ~5x more expensive than Sonnet |

**Recommendation:** Use `claude-sonnet-4-6` for production. During development and testing, switch to `claude-haiku-4-5` to reduce costs while testing workflow logic. Switch back to `claude-sonnet-4-6` before going live.

To change the model, edit the `model` parameter in the Code nodes that build the API payload:

```javascript
// In "Code: Build API Payload" and "Code: Build Disclosure Analysis Payload"
model: 'claude-haiku-4-5',  // for testing
// model: 'claude-sonnet-4-6',  // for production
```

---

## 6. Prompt Caching

Both FOI workflows use Anthropic's **prompt caching** feature to significantly reduce costs and improve response speed for the large system prompts.

### How prompt caching works

When you include `cache_control: {"type": "ephemeral"}` in a content block, Anthropic caches the processed representation of that content for up to 5 minutes. Subsequent requests that include the same cached content pay only the cache read price (~10x cheaper than re-processing) instead of the full input token price.

**For the FOI Agent:**
- The system prompt (which includes the ICB knowledge base summary) is ~1,500 tokens
- Without caching: each request re-processes these 1,500 tokens at the full input token rate
- With caching: after the first request, subsequent requests pay only the cache read rate for those tokens
- At 2-minute polling intervals, the cache is still warm for back-to-back requests

### How it is configured in the workflows

The system prompt content block includes `cache_control`:

```javascript
system: [
  {
    type: 'text',
    text: systemPrompt,  // ~1,500 tokens of ICB knowledge base
    cache_control: { type: 'ephemeral' }
  }
]
```

The `anthropic-beta: prompt-caching-2024-07-31` header is also included in the HTTP Request nodes to enable the feature.

### Monitoring cache performance

Claude API responses include usage statistics. The N8N Code nodes log these:

```javascript
console.log(`Token usage — Input: ${usage.input_tokens}, Output: ${usage.output_tokens}, Cache read: ${usage.cache_read_input_tokens || 0}`);
```

View these logs in N8N's execution history (Executions → select an execution → view the Code node output).

A high `cache_read_input_tokens` value indicates the cache is working effectively.

### When to extend the cache

If your knowledge base grows significantly (e.g. you add a large disclosure log extract to the system prompt), consider splitting the prompt into:
1. A stable "base" block (the ICB profile) — cache this
2. A dynamic block (the specific request context) — do not cache this

This gives the maximum cache benefit while keeping the dynamic content fresh.

---

## 7. Data Sovereignty Considerations for the Claude API

When the FOI Agent sends an email to the Claude API, the email content (including any personal data it may contain) is transmitted to Anthropic's servers for processing.

**Key facts:**
- Anthropic processes data in the United States
- Anthropic's data processing agreement (available at [anthropic.com/legal/data-processing-agreement](https://anthropic.com/legal/data-processing-agreement)) confirms that API inputs are not used to train models (unlike Claude.ai free tier)
- The DPA provides GDPR-compliant data transfer mechanisms (Standard Contractual Clauses)

**For NHS FOI use:**
- FOI requests are OFFICIAL classification (not OFFICIAL-SENSITIVE)
- FOI requests do not typically contain patient data
- Processing FOI request text via the Claude API under a signed DPA is defensible for OFFICIAL data

**Exception:** If an FOI request contains personal data (e.g. a requester asking for information about a named individual), the email content may contain that individual's name. Before sending to the API, consider whether the name needs to be redacted or pseudonymised. The current workflow does not automatically redact — this is a deliberate design choice for simplicity, but should be reviewed with your IG team.

See the companion technical article for the full data sovereignty framework.

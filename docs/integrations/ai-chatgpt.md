---
title: AI and ChatGPT Integration
sidebar_position: 8
description: Complete guide to integrating AI and ChatGPT with EmailEngine for email processing, summarization, and conversational search
---

# AI and ChatGPT Integration

Learn how to enhance your email workflows with artificial intelligence using EmailEngine's OpenAI/ChatGPT integration capabilities.

## Overview

EmailEngine integrates with OpenAI's API to provide AI-powered email processing capabilities:

- **Email Summarization**: Generate concise summaries of incoming emails
- **Sentiment Analysis**: Detect positive, neutral, or negative sentiment
- **Event Extraction**: Identify events and dates mentioned in emails
- **Action Items**: Extract tasks and due dates
- **Fraud Detection**: Assess risk of scam or phishing emails
- **Reply Detection**: Identify if sender expects a response
- **Conversational Search**: Ask questions about your email history

:::tip Looking for agent access instead?
This page is about EmailEngine calling a model to process incoming mail. If you want the opposite - an AI assistant calling EmailEngine to search, read and send mail on demand - see [MCP for AI Agents](/docs/mcp).
:::

### OpenAI API Access

- **Paid Account**: Recommended for production (free accounts have strict rate limits)

## Feature 1: Email Processing and Summarization

### Enable AI Processing

1. Navigate to **Configuration** → **AI Processing** in EmailEngine
2. Enter your OpenAI API key
3. Check "Enable AI Email Processing"
4. Select a model from the dropdown
5. Save configuration

![AI Processing configuration page](/img/screenshots/ai-config.png)
_The AI Configuration section with the enable checkbox, API key field and model dropdown_

### Model Selection

The dropdown is populated from your own API key: **Refresh Models** calls OpenAI's model listing endpoint and stores what came back, so the choices are whatever that key can actually use. Until the first refresh, the dropdown offers a small built-in list, currently GPT-5 Mini, GPT-5, and GPT-5 Nano.

The default is `gpt-5-mini`, and it is the right starting point for this workload: email processing is short-context classification and summarization rather than deep reasoning, so a larger model mostly buys latency and cost. Move up only if the summaries or the extracted fields are visibly worse than you need, and compare on your own mail rather than on the model's benchmark reputation.

Because the list comes from OpenAI, a model named here can be retired and a better one can appear without this page changing. Check what the dropdown offers rather than planning around a specific name.

### How It Works

When AI processing is enabled, EmailEngine automatically processes every new email arriving in the INBOX folder:

1. Email arrives in monitored account
2. EmailEngine extracts text content
3. Content sent to OpenAI API for analysis
4. Analysis results added to webhook payload

### Webhook Enhancement

With AI processing enabled, `messageNew` webhooks include additional sections:

```json
{
  "account": "example",
  "event": "messageNew",
  "data": {
    "id": "AAAAGQAACeE",
    "from": {
      "name": "Jane Doe",
      "address": "jane@example.com"
    },
    "subject": "Project meeting tomorrow at 2pm",
    "summary": {
      "id": "chatcmpl-7IzVIEp5UL3hdQ3aZJ8AHyrJrt3R0",
      "tokens": 245,
      "model": "gpt-5-mini",
      "sentiment": "positive",
      "summary": "Request to attend project meeting tomorrow at 2pm in conference room A to discuss Q4 roadmap.",
      "shouldReply": true,
      "events": [
        {
          "description": "Project meeting",
          "startTime": "2023-06-07T14:00:00"
        }
      ],
      "actions": [
        {
          "description": "Attend project meeting",
          "dueDate": "2023-06-07"
        }
      ]
    },
    "riskAssessment": {
      "risk": 1,
      "assessment": "Sender information matches and authentication checks have passed."
    }
  }
}
```

### Extracted Information

#### 1. Content Summary

Condensed version of email content (sentence or short paragraph):

```json
{
  "summary": "Request to contribute 2 to 5 euros for flower bouquets for choir teachers and concertmaster."
}
```

#### 2. Sentiment Assessment

One-word sentiment evaluation:

- **positive**: Friendly, enthusiastic, grateful
- **neutral**: Informational, factual
- **negative**: Complaint, frustration, anger

```json
{
  "sentiment": "positive"
}
```

#### 3. Reply Expectation

Boolean flag indicating if sender expects a response:

```json
{
  "shouldReply": true
}
```

#### 4. Events List

Events with dates mentioned in the email:

```json
{
  "events": [
    {
      "description": "Flower bouquets for choir teachers",
      "startTime": "2023-05-22"
    },
    {
      "description": "End of year celebration",
      "startTime": "2023-06-15",
      "endTime": "2023-06-15T18:00:00"
    }
  ]
}
```

#### 5. Actions List

Tasks recipient is expected to perform:

```json
{
  "actions": [
    {
      "description": "Contribute 2 to 5 euros for flower bouquets",
      "dueDate": "2023-05-22"
    },
    {
      "description": "RSVP for end of year celebration",
      "dueDate": "2023-06-10"
    }
  ]
}
```

#### 6. Fraud Risk Assessment

Risk score from 1 to 5 (5 being highest risk) with explanation:

```json
{
  "riskAssessment": {
    "risk": 4,
    "assessment": "Email contains urgent request for money transfer and sender domain doesn't match claimed identity. Possible phishing attempt."
  }
}
```

**Note**: AI is good at detecting scams but less effective with spam.

#### 7. Reply/Forward Text Extraction

For reply emails, removes threaded content leaving only new text:

```json
{
  "replyText": "Thanks for the update! I'll review the document and get back to you by Friday."
}
```

### Metadata Storage

#### Token Usage

The `tokens` field shows OpenAI API tokens consumed:

```json
{
  "summary": {
    "tokens": 2060,
    "model": "gpt-5-mini"
  }
}
```

Use this to track API costs and usage.

#### Request ID

The `id` field contains the OpenAI request ID for troubleshooting:

```json
{
  "summary": {
    "id": "chatcmpl-7IzVIEp5UL3hdQ3aZJ8AHyrJrt3R0"
  }
}
```

### Custom Prompts

Customize the AI analysis by modifying the system prompt:

1. Go to **Configuration** → **AI Processing**
2. Scroll to **AI Instructions** section
3. Edit the AI Prompt
4. Add custom instructions
5. Save configuration

![AI Instructions prompt editor](/img/screenshots/ai-prompt-editor.png)
_The AI Instructions section holds the editable system prompt_

#### Example: Add Language Detection

Add this line to the prompt:

```
- Return the ISO language code of the primary language used in the email as the "language" property
```

Result in webhook:

```json
{
  "summary": {
    "language": "en",
    "summary": "Meeting invitation for tomorrow at 2pm"
  }
}
```

#### Example: Custom Classification

Add business-specific classification:

```
- Classify the email type as "inquiry", "complaint", "order", or "other" in the "emailType" property
```

Result:

```json
{
  "summary": {
    "emailType": "inquiry",
    "summary": "Customer asking about product availability"
  }
}
```

### Settings Reference

Everything on the **Configuration > AI Processing** page is also settable through the [Settings API](/docs/api/post-v-1-settings):

| Setting | Purpose |
|---------|---------|
| `openAiAPIKey` | API key. Required before any AI processing runs |
| `generateEmailSummary` | Turn on summaries, sentiment, events, actions, and risk assessment |
| `openAiGenerateEmbeddings` | Turn on embedding generation |
| `openAiModel` | Model name, for example `gpt-5-mini` |
| `openAiPrompt` | The system prompt, as edited above |
| `openAiAPIUrl` | Base URL of the API. Point this at Azure OpenAI or an OpenAI-compatible gateway |
| `openAiTemperature` | Sampling temperature |
| `openAiTopP` | Nucleus sampling cutoff |
| `openAiMaxTokens` | Cap on tokens per request |
| `openAiPreProcessingFn` | JavaScript filter deciding which messages are worth processing, see [below](#ai-pre-processing-filter-openaipreprocessingfn) |

Lowering `openAiMaxTokens` truncates long messages before they reach the model, which is the most direct lever on cost. `openAiPreProcessingFn` is the more selective one, since a message it rejects costs nothing at all.

### Handling Failures

EmailEngine skips AI processing if:

- OpenAI API request fails
- Rate limit exceeded
- Timeout occurs
- Email has no text content

In these cases, the `summary` section is omitted from webhook payload. Check EmailEngine logs for details.

### Webhook Content Configuration

If webhooks are configured not to include email content, AI summarization may fail (nothing to summarize). Ensure webhook configuration includes at least text content.

## Use Cases and Applications

Everything below is driven from the enriched `messageNew` payload. Once AI processing is on, your webhook handler branches on the fields rather than calling any additional endpoint:

```javascript
app.post('/webhook', async (req, res) => {
  res.json({ success: true });

  const { event, data } = req.body;
  if (event !== 'messageNew' || !data.summary) return;

  const { summary, riskAssessment } = data;

  // Fraud triage: risk runs 1 to 5
  if (riskAssessment?.risk >= 4) {
    return quarantine(data.id, riskAssessment.assessment);
  }

  // Tasks and calendar entries the model found in the body
  for (const action of summary.actions || []) {
    await createTask({ title: action.description, dueDate: action.dueDate });
  }

  for (const ev of summary.events || []) {
    await createCalendarEvent({ title: ev.description, start: ev.startTime, end: ev.endTime });
  }

  // Support triage: an unhappy sender who expects an answer goes to the front
  if (summary.sentiment === 'negative' && summary.shouldReply) {
    await escalate(data.id);
  }
});
```

:::note `riskAssessment` sits next to `summary`, not inside it
EmailEngine lifts the risk assessment out of the summary object before sending the webhook, so read `data.riskAssessment` rather than `data.summary.riskAssessment`.
:::

Which field drives which workflow:

| Field | Typical use |
|-------|-------------|
| `summary.sentiment` | Support triage, escalating negative mail |
| `summary.shouldReply` | Priority inbox, SLA timers, follow-up reminders |
| `summary.actions[]` | Creating tasks with a `description` and `dueDate` |
| `summary.events[]` | Creating calendar entries from `startTime` and `endTime` |
| `riskAssessment.risk` | Fraud and phishing quarantine, 1 to 5 |
| `replyText` | Storing just the new text of a reply, without the quoted thread |

The model does not always populate every field. Treat each one as optional and fall back to your existing routing when it is missing, since an OpenAI outage or a rate limit leaves the message delivered but unenriched. See [Handling Failures](#handling-failures).

### 6. Smart Email Search Assistant

:::danger Being removed on October 1, 2026
The `POST /v1/chat/{account}` endpoint is part of the Document Store feature, which is deprecated and **will be removed from EmailEngine releases starting October 1, 2026**. After that release, syncing to Elasticsearch, chat with email, and unified search are gone. Do not build new integrations on this endpoint.

Until then it is disabled by default: since v2.71.0 the endpoint returns `404` unless you enable the startup gate (`EENGINE_DOCUMENT_STORE_ENABLED=true`) in addition to turning on Document Store (Elasticsearch) indexing and the "Chat with emails" feature. It is also left out of the API reference on this site, though EmailEngine's own [OpenAPI document](/docs/api-reference/openapi-spec) still describes it.
:::

Build conversational email search for users:

**cURL:**

```bash
curl -X POST "https://emailengine.example.com/v1/chat/user123" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "Did I receive the invoice from Acme Corp?"
  }'
```

**Response:**

```json
{
  "success": true,
  "answer": "Yes, you received an invoice from Acme Corp on October 5th for $1,500.",
  "messages": [
    {
      "id": "AAAAGQAACeE",
      "from": {
        "name": "Acme Corp",
        "address": "billing@acmecorp.com"
      },
      "subject": "Invoice #12345",
      "date": "2023-10-05T10:00:00.000Z"
    }
  ]
}
```

```bash
curl -X POST "https://emailengine.example.com/v1/chat/user123" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "question": "Did I receive the invoice from Acme Corp?" }'
```

The response carries the generated `answer` along with the `messages` it was drawn from, so you can link the reader back to the source email rather than asking them to trust the summary.

## Privacy and Compliance

### Data Processing

**Important**: When OpenAI integration is enabled, EmailEngine uploads email text content to OpenAI servers.

**OpenAI Policy**: OpenAI does not use API data for training models.

**Your Responsibility**: Verify this behavior complies with:

- User data processing agreements
- GDPR requirements
- Industry-specific regulations (HIPAA, etc.)
- Company privacy policies

### User Consent

Recommendations:

1. **Transparent Disclosure**: Inform users that AI processes their emails
2. **Opt-In**: Allow users to enable/disable AI processing
3. **Data Retention**: Clarify how long AI-processed data is stored
4. **Third-Party Processing**: Disclose data sent to OpenAI

### Per-Account Control

You can enable/disable AI processing per account if needed (would require custom implementation).

## AI Pre-Processing Filter (openAiPreProcessingFn)

By default, AI processing is applied to all incoming emails in the INBOX folder. The `openAiPreProcessingFn` setting allows you to define a JavaScript function that filters which emails get processed by the AI, giving you fine-grained control over AI usage and costs.

### How It Works

When configured, the pre-processing filter runs before any AI processing occurs:

1. New email arrives in monitored account
2. Pre-processing filter function evaluates the email
3. If function returns `true`, email is sent to OpenAI for analysis
4. If function returns any other value, AI processing is skipped

### Configuration

Configure the filter via the Settings API:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "openAiPreProcessingFn": "// Only process emails from specific domains\nconst senderDomain = payload.data.from?.address?.split(\"@\")[1];\nif ([\"important-client.com\", \"vip-customer.org\"].includes(senderDomain)) {\n  return true;\n}\nreturn false;"
  }'
```

### Filter Function Structure

The filter function receives the webhook payload and must return `true` to allow AI processing:

```javascript
// payload contains the full messageNew webhook data
// Return true to process with AI, false to skip

// Skip automated emails
if (payload.data.headers && payload.data.headers["auto-submitted"]) {
  return false;
}

// Skip emails from no-reply addresses
if (payload.data.from?.address?.includes("noreply")) {
  return false;
}

// Process all other emails
return true;
```

### Example Filters

**1. Process Only High-Priority Senders**

```javascript
// Only AI-process emails from VIP domains
const vipDomains = ["client.com", "partner.org", "executive.net"];
const senderDomain = payload.data.from?.address?.split("@")[1]?.toLowerCase();

if (vipDomains.includes(senderDomain)) {
  return true;
}
return false;
```

**2. Skip Newsletters and Automated Messages**

```javascript
// Skip common newsletter and automated email patterns
const from = payload.data.from?.address?.toLowerCase() || "";
const subject = payload.data.subject?.toLowerCase() || "";

// Skip newsletters
if (from.includes("newsletter") || from.includes("digest")) {
  return false;
}

// Skip automated messages
if (payload.data.headers?.["auto-submitted"] || payload.data.headers?.["x-auto-response-suppress"]) {
  return false;
}

// Skip common notification subjects
if (subject.includes("notification") || subject.includes("alert")) {
  return false;
}

return true;
```

**3. Process Based on Subject Keywords**

```javascript
// Only process emails with specific keywords
const subject = payload.data.subject?.toLowerCase() || "";
const keywords = ["urgent", "invoice", "contract", "proposal", "meeting"];

for (const keyword of keywords) {
  if (subject.includes(keyword)) {
    return true;
  }
}
return false;
```

**4. Size-Based Filtering**

```javascript
// Skip very short or very long emails (likely spam or bulk)
const textSize = payload.data.text?.encodedSize?.plain || 0;

if (textSize < 50) {
  // Too short - likely spam
  return false;
}
if (textSize > 50000) {
  // Too long - will consume many tokens
  return false;
}
return true;
```

**5. Time-Based Processing**

```javascript
// Only process recent emails (skip old backlog)
const emailDate = new Date(payload.data.date);
const now = new Date();
const hoursDiff = (now - emailDate) / (1000 * 60 * 60);

// Skip emails older than 24 hours
if (hoursDiff > 24) {
  return false;
}
return true;
```

### Available Payload Data

The filter function has access to the full `messageNew` webhook payload:

```javascript
payload.account;           // Account ID
payload.path;              // Mailbox path (e.g., "INBOX")
payload.data.id;           // Message ID
payload.data.from;         // { name, address }
payload.data.to;           // [{ name, address }, ...]
payload.data.subject;      // Email subject
payload.data.date;         // Message date
payload.data.headers;      // Email headers object
payload.data.text;         // { encodedSize: { plain, html } }
payload.data.attachments;  // Attachment metadata array
```

### Execution Environment

The filter function runs in the same execution context as other pre-processing functions:

**Available:**

- Standard JavaScript (ES6+)
- `Date`, `Math`, `JSON`, `RegExp`
- `fetch` - Make HTTP requests (a wrapped fetch implementation is injected)
- `env` - Script environment variables (from `scriptEnv` setting)
- `logger` - Pino.js logger for debugging

**Not injected as globals:**

- `require()` - modules are not provided
- Filesystem or system helpers are not provided

:::warning Not a security sandbox
Filter and pre-processing functions run on Node's `vm` module, which is an isolation convenience, **not** a hardened security boundary - code executed here can reach the host process and runs with full server privileges. Only enable and author functions you fully trust; never expose function authoring to untrusted users. See [Execution Environment](/docs/advanced/pre-processing#execution-environment).
:::

### Debugging Filters

Errors in the filter function are logged and the email is skipped (treated as returning `false`). Check EmailEngine logs for filter errors:

```bash
# View filter-related log entries
journalctl -u emailengine | grep "llm-pre-process"
```

You can also use the logger for debugging:

```javascript
logger.info({ from: payload.data.from?.address, subject: payload.data.subject, msg: "Evaluating email" });

const result = payload.path === "INBOX";
logger.info({ result, msg: "Filter decision" });

return result;
```

### Best Practices

1. **Start broad, then narrow** - Begin with minimal filtering and add rules as needed
2. **Monitor costs** - Track token usage to measure filter effectiveness
3. **Log decisions** - Use `logger` to track why emails are filtered
4. **Handle missing data** - Use optional chaining (`?.`) for potentially undefined values
5. **Keep it fast** - Complex logic adds processing overhead

## Cost Management

### Estimating Costs

OpenAI charges based on token usage. Pricing varies by model and changes over time. Check [OpenAI's pricing page](https://openai.com/pricing) for current rates.

**General pricing tiers:**

- **Mini/Nano models** (GPT-5 Mini, GPT-4o Mini) - Lowest cost, best for high-volume processing
- **Standard models** (GPT-5, GPT-4o) - Moderate cost, balanced performance
- **Pro models** (GPT-5 Pro, O3 Pro) - Higher cost, maximum capability

**Example**: Processing 100 emails per day with GPT-5 Mini:

- Average email: ~500 tokens input, 200 tokens output
- Daily tokens: ~70,000 tokens
- Monthly cost varies by current pricing - typically very affordable with mini models

### Cost Optimization

1. **Use Mini Models**: GPT-5 Mini or GPT-4o Mini for most email processing
2. **Filter Emails**: Only process emails from Inbox (skip spam, notifications)
3. **Selective Processing**: Process only emails matching certain criteria
4. **Monitor Usage**: Track `tokens` field in webhooks
5. **Rate Limiting**: Limit processing during high-volume periods

### Monitoring Token Usage

Every enriched `messageNew` payload reports what the call cost, so metering needs no separate bookkeeping against OpenAI:

```javascript
app.post('/webhook', (req, res) => {
  res.json({ success: true });

  const { summary } = req.body.data || {};
  if (!summary) return; // AI processing off, skipped, or failed

  metrics.increment('openai.tokens', summary.tokens, {
    model: summary.model,
    account: req.body.account
  });
});
```

Aggregating by `account` shows which mailboxes drive the spend, which is usually a small number of high-volume ones. See [Cost Optimization](#cost-optimization) for narrowing what gets processed.

## See Also

- [MCP for AI agents](/docs/mcp) - The opposite direction: an agent calling EmailEngine
- [Pre-processing functions](/docs/advanced/pre-processing) - Filtering which messages reach a model
- [Webhooks overview](/docs/webhooks/overview) - Where the enriched payload arrives
- [Compliance and data handling](/docs/deployment/compliance) - What leaves the instance when AI processing is on

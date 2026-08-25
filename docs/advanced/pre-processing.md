---
title: Pre-Processing Functions
sidebar_position: 6
description: Use JavaScript-based pre-processing functions to filter and transform webhooks and messages in EmailEngine
keywords:
  - pre-processing
  - javascript
  - webhooks
  - filtering
  - transformation
  - custom logic
---

# Pre-Processing Functions

Pre-processing functions allow you to run custom JavaScript code to filter or transform data before EmailEngine processes it. Use these functions to implement custom business logic, filter unwanted events, or modify webhook payloads.

## Overview

EmailEngine supports pre-processing for:

- **Webhooks** - Filter or modify webhook payloads before delivery
- **Custom Routes** - Transform data before processing

Pre-processing functions are small JavaScript snippets that run in an isolated execution context with a restricted set of globals and an enforced timeout.

## Function Types

:::warning Code Format
Pre-processing code is **function body content**, not a full function definition. The code runs in the main scope and must return a value directly. The webhook payload is available as `payload`.

```javascript
if (payload.path === "INBOX") {
  return true;
}
return false;
```

:::

### 1. Filter Functions

Filter functions determine whether an event should be processed or discarded.

**Return value:**

- `true` - Process the event (must be exactly `true`, not just truthy)
- `false` or any other value - Discard the event
- Exception thrown - Discard the event

**Use cases:**

- Skip webhooks for automated messages
- Filter out spam or promotional emails

**Example - Skip auto-reply emails:**

```javascript
// Return true to send webhook, false to skip

// Skip auto-replies
if (payload.data.headers && payload.data.headers["auto-submitted"]) {
  return false;
}

// Skip out-of-office messages
if (payload.data.subject && /out of office/i.test(payload.data.subject)) {
  return false;
}

// Process all other webhooks
return true;
```

### 2. Mapping Functions

Mapping functions modify the structure or content of data before processing.

**Return value:**

- Modified data object
- Original data unchanged if no return value

**Use cases:**

- Add custom fields to webhooks
- Redact sensitive information
- Normalize data formats
- Enrich with additional context

**Example - Add custom fields:**

```javascript
// Add custom tracking ID
payload.customId = `${payload.account}-${Date.now()}`;

// Add priority based on subject
if (payload.data.subject && /urgent/i.test(payload.data.subject)) {
  payload.priority = "high";
} else {
  payload.priority = "normal";
}

// Redact sensitive content
if (payload.data.text && payload.data.text.plain) {
  payload.data.text.plain = payload.data.text.plain.replace(/ssn:\s*\d{3}-\d{2}-\d{4}/gi, "ssn: [REDACTED]");
}

return payload;
```

## Configuration

### Webhook Pre-Processing

Configure pre-processing for webhook routes in the EmailEngine UI.

#### Step 1: Open the webhook route

1. Go to **Integrations** → **Webhook Routes**
2. Click on the webhook route to configure
3. Scroll to **Pre-Processing Function** section

#### Step 2: Add Filter Function

```javascript
// Filter: return true to send webhook, false to skip

// Only send webhooks for inbox messages
if (payload.path !== "INBOX") {
  return false;
}

// Skip notifications (usually automated)
if (payload.data.from && /noreply|no-reply/i.test(payload.data.from.address)) {
  return false;
}

// Skip old messages (older than 1 hour)
if (payload.data.date) {
  const messageAge = Date.now() - new Date(payload.data.date).getTime();
  if (messageAge > 3600000) {
    // 1 hour in milliseconds
    return false;
  }
}

return true;
```

#### Step 3: Add Mapping Function

```javascript
// Mapping: modify payload and return it

// Add custom fields
payload.metadata = {
  receivedAt: new Date().toISOString(),
  environment: "production",
  version: "1.0",
};

// Categorize by subject
if (payload.data.subject) {
  const subject = payload.data.subject.toLowerCase();
  if (subject.includes("invoice") || subject.includes("payment")) {
    payload.category = "billing";
  } else if (subject.includes("support") || subject.includes("help")) {
    payload.category = "support";
  } else {
    payload.category = "general";
  }
}

// Extract ticket ID from subject
const ticketMatch = payload.data.subject && payload.data.subject.match(/#(\d+)/);
if (ticketMatch) {
  payload.ticketId = ticketMatch[1];
}

return payload;
```

#### Step 4: Test Function

Use the **Set test payload** button to provide sample data for testing your function. The editor runs your filter and mapping functions in the browser and shows:

- **Evaluation result** - Whether the filter function returns `true` (matches) or not
- **Mapping preview** - The transformed payload after the mapping function runs

You can select from predefined example payloads or enter custom JSON data to test different scenarios.

## Common Use Cases

### 1. Skip Automated Emails

```javascript
// Check Auto-Submitted header
if (payload.data.headers && payload.data.headers["auto-submitted"] && payload.data.headers["auto-submitted"][0] !== "no") {
  return false;
}

// Check for common automated addresses
const automatedPatterns = [/noreply/i, /no-reply/i, /donotreply/i, /notifications?/i, /mailer-daemon/i, /postmaster/i];

if (payload.data.from && payload.data.from.address) {
  for (const pattern of automatedPatterns) {
    if (pattern.test(payload.data.from.address)) {
      return false;
    }
  }
}

// Check subject for automated patterns
const automatedSubjects = [/out of office/i, /automatic reply/i, /auto-reply/i, /mail delivery fail/i];

if (payload.data.subject) {
  for (const pattern of automatedSubjects) {
    if (pattern.test(payload.data.subject)) {
      return false;
    }
  }
}

return true;
```

### 2. Extract and Normalize Data

```javascript
// Extract email addresses from CC and BCC
const allRecipients = [...(payload.data.to || []), ...(payload.data.cc || []), ...(payload.data.bcc || [])].map((r) => r.address);

payload.allRecipients = [...new Set(allRecipients)]; // Remove duplicates

// Parse subject line for ticket/order numbers
if (payload.data.subject) {
  // Extract ticket ID (e.g., "#12345", "TICKET-12345")
  const ticketMatch = payload.data.subject.match(/#(\d+)|TICKET-(\d+)/i);
  if (ticketMatch) {
    payload.ticketId = ticketMatch[1] || ticketMatch[2];
  }

  // Extract order ID (e.g., "Order #12345", "Order ID: 12345")
  const orderMatch = payload.data.subject.match(/order\s*#?:?\s*(\d+)/i);
  if (orderMatch) {
    payload.orderId = orderMatch[1];
  }
}

// Normalize sender domain
if (payload.data.from && payload.data.from.address) {
  const domain = payload.data.from.address.split("@")[1];
  payload.senderDomain = domain.toLowerCase();

  // Flag internal emails
  payload.isInternal = ["example.com", "company.com"].includes(domain);
}

return payload;
```

### 3. Priority and Categorization

```javascript
// Determine priority
payload.priority = "normal";

// High priority indicators
const urgentKeywords = ["urgent", "asap", "important", "critical"];
const subject = (payload.data.subject || "").toLowerCase();

for (const keyword of urgentKeywords) {
  if (subject.includes(keyword)) {
    payload.priority = "high";
    break;
  }
}

// VIP senders
const vipDomains = ["important-client.com", "executive.com"];
if (payload.data.from && payload.data.from.address) {
  const domain = payload.data.from.address.split("@")[1];
  if (vipDomains.includes(domain)) {
    payload.priority = "high";
    payload.isVip = true;
  }
}

// Categorize by content
const categories = {
  billing: ["invoice", "payment", "receipt", "billing"],
  support: ["support", "help", "question", "issue"],
  sales: ["quote", "proposal", "pricing", "demo"],
  hr: ["benefits", "payroll", "pto", "vacation"],
};

for (const [category, keywords] of Object.entries(categories)) {
  for (const keyword of keywords) {
    if (subject.includes(keyword)) {
      payload.category = category;
      break;
    }
  }
  if (payload.category) break;
}

payload.category = payload.category || "general";

return payload;
```

### 4. Redact Sensitive Information

```javascript
// Patterns for sensitive data
const patterns = {
  ssn: /\b\d{3}-\d{2}-\d{4}\b/g,
  creditCard: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g,
  phone: /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g,
  email: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
};

// Redact from text (note: text content may be in payload.data.text.plain)
if (payload.data.text && payload.data.text.plain) {
  payload.data.text.plain = payload.data.text.plain.replace(patterns.ssn, "SSN:[REDACTED]");
  payload.data.text.plain = payload.data.text.plain.replace(patterns.creditCard, "CARD:[REDACTED]");
}

// Flag as containing sensitive data
const originalText = (payload.data.text && payload.data.text.plain) || "";
if (patterns.ssn.test(originalText) || patterns.creditCard.test(originalText)) {
  payload.containsSensitiveData = true;
}

return payload;
```

### 5. Add Metadata and Context

```javascript
// Add processing metadata
payload.processing = {
  receivedAt: new Date().toISOString(),
  version: "2.0",
};

// Count attachments by type
if (payload.data.attachments) {
  payload.attachmentStats = {
    total: payload.data.attachments.length,
    images: payload.data.attachments.filter((a) => a.contentType?.startsWith("image/")).length,
    documents: payload.data.attachments.filter((a) => a.contentType?.includes("pdf") || a.contentType?.includes("word")).length,
    totalSize: payload.data.attachments.reduce((sum, a) => sum + (a.encodedSize || 0), 0),
  };
}

return payload;
```

## Available Data

### Webhook Payload

Pre-processing functions receive the complete webhook payload. The payload has a nested structure with top-level webhook metadata and message details inside the `data` property:

```javascript
{
  // Top-level webhook metadata
  serviceUrl: "https://emailengine.example.com",
  account: "testaccount",
  date: "2024-10-13T14:23:45.678Z",
  path: "INBOX",
  specialUse: "\\Inbox",
  event: "messageNew",

  // Message data (nested inside `data`)
  data: {
    id: "AAAABgAAAdk",
    uid: 123,
    path: "INBOX",
    date: "2024-10-13T14:20:00.000Z",
    flags: [],
    unseen: true,
    subject: "Important Message",
    from: {
      name: "John Doe",
      address: "john@example.com"
    },
    to: [
      { name: "Jane Smith", address: "jane@example.com" }
    ],
    replyTo: [],
    messageId: "<abc123@example.com>",
    headers: {
      "return-path": ["<john@example.com>"],
      "message-id": ["<abc123@example.com>"],
      "from": ["John Doe <john@example.com>"],
      "to": ["Jane Smith <jane@example.com>"],
      "subject": ["Important Message"],
      "content-type": ["text/plain; charset=utf-8"],
      "auto-submitted": ["no"]  // Present for automated messages
    },
    text: {
      id: "...",
      encodedSize: { plain: 1234, html: 5678 }
    },
    attachments: [
      {
        id: "abc123",
        contentType: "application/pdf",
        encodedSize: 12345,
        filename: "document.pdf",
        embedded: false,
        inline: false
      }
    ]
  }
}
```

**Accessing data in filter/mapping functions:**

```javascript
// Top-level properties (webhook metadata)
payload.event; // "messageNew"
payload.account; // "testaccount"
payload.path; // "INBOX"

// Message properties (inside payload.data)
payload.data.subject; // "Important Message"
payload.data.from; // { name: "...", address: "..." }
payload.data.headers; // { "auto-submitted": [...], ... }
payload.data.attachments; // [...]
```

## Execution Environment

Pre-processing functions run in an isolated execution context (built on Node's `vm` module) with a restricted set of globals and an enforced timeout. Note that Node's `vm` is an isolation mechanism, not a hardened security boundary - only run pre-processing code you trust.

The context exposes a limited set of capabilities:

**Available:**

- Standard JavaScript (ES6+)
- Top-level `await` is supported
- `Date`, `Math`, `JSON`, `RegExp`
- `fetch` - Make HTTP requests to external services
- `URL` - URL parsing and manipulation
- `logger` - Pino.js logger instance (logs to EmailEngine's stdout)
- `env` - The `scriptEnv` settings object (configure via Settings API)

**Not Available:**

- `require()` - Cannot import modules
- `fs` - No filesystem access
- `process`, `child_process` - No system access

**Using `env` for Configuration:**

Store sensitive values like API keys in `scriptEnv` settings rather than hardcoding them:

```javascript
// Access values from scriptEnv settings
const apiKey = env.MY_API_KEY;
const webhookSecret = env.WEBHOOK_SECRET;

if (apiKey) {
  // Use the API key
  const response = await fetch("https://api.example.com/validate", {
    headers: { Authorization: `Bearer ${apiKey}` },
  });
}

return true;
```

## Script Environment Variables (scriptEnv)

The `scriptEnv` setting allows you to pass secrets, API keys, and configuration values to pre-processing scripts without hardcoding them. This is the recommended approach for managing sensitive data in scripts.

### Configuring scriptEnv

Configure `scriptEnv` via the Settings API. The value must be a JSON string containing key-value pairs:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scriptEnv": "{\"MY_API_KEY\":\"your-api-key-here\",\"WEBHOOK_SECRET\":\"your-secret\",\"SLACK_WEBHOOK_URL\":\"https://hooks.slack.com/...\"}"
  }'
```

Or with formatted JSON for readability:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scriptEnv": "{\"API_KEY\":\"sk-abc123\",\"ENVIRONMENT\":\"production\",\"DEBUG_MODE\":\"false\"}"
  }'
```

### Accessing Environment Variables in Scripts

The `env` object is available globally in all pre-processing scripts:

```javascript
// Access a single variable
const apiKey = env.API_KEY;

// Check if variable exists
if (env.SLACK_WEBHOOK_URL) {
  await fetch(env.SLACK_WEBHOOK_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      text: `New email from ${payload.data.from.address}`,
    }),
  });
}

// Use environment to control behavior
if (env.ENVIRONMENT === "production") {
  // Production-specific logic
}

return true;
```

### Use Cases

**1. External API Integration**

```javascript
// Call an external classification API
const response = await fetch(env.CLASSIFICATION_API_URL, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${env.CLASSIFICATION_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    subject: payload.data.subject,
    from: payload.data.from.address,
  }),
});

const result = await response.json();
payload.classification = result.category;
return payload;
```

**2. Webhook Secret Validation**

```javascript
// Add a shared secret to webhook payloads
payload.webhookSecret = env.WEBHOOK_SECRET;
payload.timestamp = Date.now();
return payload;
```

**3. Environment-Specific Filtering**

```javascript
// Different behavior per environment
if (env.ENVIRONMENT === "development") {
  // In development, process all emails
  return true;
}

// In production, only process inbox emails
return payload.path === "INBOX";
```

**4. Feature Flags**

```javascript
// Toggle features via configuration
if (env.ENABLE_SLACK_NOTIFICATIONS === "true") {
  await fetch(env.SLACK_WEBHOOK_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text: `Email received: ${payload.data.subject}` }),
  });
}
return true;
```

### Best Practices

1. **Never hardcode secrets** - Always use `scriptEnv` for API keys, tokens, and passwords
2. **Use descriptive names** - Use clear, uppercase names like `OPENAI_API_KEY`, `SLACK_WEBHOOK_URL`
3. **Validate before use** - Check if variables exist before using them: `if (env.MY_KEY) { ... }`
4. **Keep it simple** - Store only values needed by scripts; use EmailEngine settings for EmailEngine configuration
5. **Update atomically** - When updating `scriptEnv`, include all values as the entire object is replaced

### Retrieving Current Configuration

Get the current `scriptEnv` value:

```bash
curl "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

The response includes `scriptEnv` (as a JSON string) among other settings

**Using `logger` for Structured Logging:**

```javascript
logger.info({ account: payload.account, path: payload.path, msg: "Processing webhook" });
logger.warn({ subject: payload.data.subject, msg: "Suspicious email detected" });

return true;
```

## Performance Considerations

### 1. Keep Functions Fast

Pre-processing runs for every event. Keep functions lightweight:

```javascript
// Fast - simple checks
return payload.path === "INBOX" && !payload.data.headers["auto-submitted"];
```

```javascript
// Slow - avoid external requests in pre-processing
const response = await fetch("https://api.openai.com/v1/chat/completions", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${env.OPENAI_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: "gpt-4",
    messages: [{ role: "user", content: `Is this spam? ${payload.data.subject}` }],
  }),
});
const result = await response.json();
return result.choices[0].message.content.includes("not spam");
```

:::tip Use Webhooks for Slow Operations
Instead of calling external APIs in pre-processing, let the webhook pass through and handle AI classification in your webhook handler. This keeps EmailEngine responsive while your application handles the slow operations asynchronously.
:::

### 2. Cache Computed Values

If checking multiple conditions, store intermediate results:

```javascript
const subject = (payload.data.subject || "").toLowerCase(); // Compute once

// Use cached value
payload.isUrgent = subject.includes("urgent") || subject.includes("important");

payload.isBilling = subject.includes("invoice") || subject.includes("payment");

return payload;
```

### 3. Exit Early

Return as soon as you know the result:

```javascript
// Exit early if conditions not met
if (payload.path !== "INBOX") return false;
if (!payload.data.from || !payload.data.from.address) return false;
if (payload.data.headers && payload.data.headers["auto-submitted"]) return false;

// Only process if all checks passed
return true;
```

## Debugging

### Using the Logger

Use `logger` (Pino.js) for debugging:

```javascript
logger.info({ account: payload.account, path: payload.path, msg: "Processing webhook" });
logger.debug({ autoSubmitted: !!payload.data.headers?.["auto-submitted"], msg: "Header check" });

const result = payload.path === "INBOX";
logger.info({ result, msg: "Filter decision" });

return result;
```

Log entries appear in EmailEngine's stdout alongside other application logs.

### Check EmailEngine Logs

EmailEngine logs to stdout. Use your process manager or Docker logs to view output:

```bash
# If running directly
node server.js 2>&1 | grep "subscript"

# If using Docker
docker logs -f emailengine 2>&1 | grep "subscript"

# If using systemd
journalctl -u emailengine -f | grep "subscript"
```

### Monitor Execution Time

Log timing information:

```javascript
const start = Date.now();

// Your transformations
payload.customField = "processed";

const duration = Date.now() - start;
logger.info({ duration, msg: "Processing completed" });

return payload;
```

## HTML Email Pre-Processing for Web Display

Not to be confused with the functions above. EmailEngine can also pre-process message HTML into a
form that is safe to inject into a web page, sanitizing the markup, inlining images, and folding
quoted thread history into a collapsible block.

That is covered on its own page: [Web-Safe HTML](/docs/receiving/web-safe-html).

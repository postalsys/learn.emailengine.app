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

Pre-processing functions are short pieces of JavaScript that EmailEngine runs against an event payload before acting on it. They decide whether a webhook route receives an event, and what the delivered body looks like. This page is the reference for the script runtime: what the code is wrapped in, what it can see, what a return value does, and how long it may run.

## Where They Run

The same runtime serves three features:

| Feature | Filter function | Map function | Configured at |
| ------- | --------------- | ------------ | ------------- |
| [Webhook routes](/docs/webhooks/webhook-routing) | Decides whether the route gets the event | Rewrites the body sent to the route's URL | **Integrations** > **Webhook Routes**, per route |
| [AI pre-processing filter](/docs/integrations/ai-chatgpt#ai-pre-processing-filter-openaipreprocessingfn) | Decides which messages are sent to the LLM | none | `openAiPreProcessingFn` setting |
| Document Store pre-processing | Decides which messages are indexed | Rewrites the indexed document | Document Store settings. Deprecated; removed from releases starting 2026-10-01 |

The rest of this page is written for webhook routes. The other two use the same globals, timeout and error handling.

## Function Format

The text you enter is a **function body**, not a function definition. EmailEngine wraps it as

```javascript
(async (payload) => {
  // your code
})(payload);
```

so `return` ends the function and `await` is available at the top level. The event is the `payload` variable.

```javascript
if (payload.path === "INBOX") {
  return true;
}
return false;
```

### Filter Functions

A filter function decides whether the event goes any further.

**Return value:**

- `true` - The route receives the event
- `false`, `undefined`, or nothing - The route skips it
- An exception - The route skips it, and the error is recorded in the route's **Error log** tab (the last 20 entries are kept)

Return a boolean. On the server any truthy value counts as a match, but the editor's live preview reports a non-boolean result as `no match` with a warning, so a filter that returns a string or an object looks broken in the UI even though it would match in production.

**Example - skip auto-replies:**

```javascript
// Skip messages that declare themselves automatic
if (payload.data.headers && payload.data.headers["auto-submitted"] && payload.data.headers["auto-submitted"][0] !== "no") {
  return false;
}

// Skip out-of-office replies by subject
if (payload.data.subject && /out of office/i.test(payload.data.subject)) {
  return false;
}

return true;
```

### Map Functions

A map function runs only when the filter matched, and shapes what is delivered.

**Return value:**

- An object - Becomes the webhook body, replacing the original payload entirely. Anything you leave out is not sent
- `undefined` or nothing - The original payload is sent unchanged
- An exception - The error is recorded in the route's **Error log** tab and the original payload is sent unchanged, as if the map had returned nothing

Each function receives its own copy of the payload, so a map function can change fields in place and return `payload`. Because the map decides the whole body, it can also return something that is not the payload at all, which is how a route posts to a service with a fixed input format, for example a Slack message.

**Example - add fields and return the payload:**

```javascript
payload.customId = `${payload.account}-${Date.now()}`;

if (payload.data.subject && /urgent/i.test(payload.data.subject)) {
  payload.priority = "high";
} else {
  payload.priority = "normal";
}

return payload;
```

**Example - build a different body:**

```javascript
return {
  text: `New message for ${payload.account}: ${payload.data.subject || "(no subject)"}`,
  from: payload.data.from && payload.data.from.address,
  messageId: payload.data.messageId
};
```

## Available Data

### The `payload` Object

The payload is the webhook event exactly as EmailEngine would deliver it, with two additions: the `data` property carries the message, and `_route` identifies the route being evaluated.

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2024-10-13T14:23:45.678Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageNew",
  "_route": {
    "id": "AAABkxEnOGIAAAAB"
  },
  "data": {
    "id": "AAAABgAAAdk",
    "uid": 123,
    "path": "INBOX",
    "date": "2024-10-13T14:20:00.000Z",
    "flags": [],
    "unseen": true,
    "size": 4127,
    "subject": "Important Message",
    "from": {
      "name": "John Doe",
      "address": "john@example.com"
    },
    "replyTo": [],
    "to": [
      {
        "name": "Jane Smith",
        "address": "jane@example.com"
      }
    ],
    "messageId": "<abc123@example.com>",
    "headers": {
      "return-path": ["<john@example.com>"],
      "message-id": ["<abc123@example.com>"],
      "from": ["John Doe <john@example.com>"],
      "to": ["Jane Smith <jane@example.com>"],
      "subject": ["Important Message"],
      "content-type": ["text/plain; charset=utf-8"],
      "auto-submitted": ["no"]
    },
    "text": {
      "id": "AAAABgAAAdmRkA",
      "encodedSize": {
        "plain": 1234,
        "html": 5678
      }
    },
    "attachments": [
      {
        "id": "AAAABgAAAdky",
        "contentType": "application/pdf",
        "encodedSize": 12345,
        "filename": "document.pdf",
        "embedded": false,
        "inline": false
      }
    ]
  }
}
```

The top-level fields are the same for every event. What `data` holds depends on the event; the [webhooks overview](/docs/webhooks/overview#webhook-events) links to the payload of each one. Header values are arrays, one entry per occurrence of the header.

`_route` is present so a script can tell which route it is running for. It is stripped from the body when the original payload is delivered, but if a map function returns the `payload` object it comes along; delete it first if the receiving service objects to unknown fields.

**Accessing data:**

```javascript
payload.event; // "messageNew"
payload.account; // "user123"
payload.path; // "INBOX"

payload.data.subject;
payload.data.from; // { name, address }
payload.data.headers["auto-submitted"]; // ["no"]
payload.data.attachments; // []
```

## Execution Environment

Functions run inside a fresh V8 context created with Node's `vm` module. The context starts with the JavaScript built-ins (`Object`, `Array`, `Date`, `Math`, `JSON`, `RegExp`, `Promise`, `Map`, `Set` and the rest of the language) and the globals below, and nothing else.

**Available:**

| Global | What it is |
| ------ | ---------- |
| `payload` | A deep copy of the event. Changes do not leak into other routes |
| `fetch(url, options)` | HTTP client, the same one EmailEngine uses internally. Connection attempts time out after 30 seconds and the response after 90 seconds (`EENGINE_FETCH_TIMEOUT` changes the latter); `429` responses and transient network errors are retried automatically |
| `URL` | The WHATWG `URL` class |
| `logger` | Writes to EmailEngine's log. `logger.trace`, `.debug`, `.info`, `.warn`, `.error` and `.fatal` each take one argument, a string or an object with a `msg` property |
| `env` | The parsed `scriptEnv` setting, see below |

**Not available:**

- `require()` and `import` - No modules
- `process`, `Buffer`, `fs`, `child_process` - No access to the host
- `console` - Use `logger`
- `setTimeout`, `setInterval`, `AbortSignal` - No timers, so there is no way to abandon a slow `fetch` from inside the script beyond the built-in timeouts above

**Time limit:** 30 seconds of synchronous execution. The limit interrupts a loop that never yields, but it does not bound time spent waiting on `await`. A script that waits on a slow `fetch` runs until that request's own timeout fires, which is why external calls belong in the webhook handler rather than in the filter (see [Performance](#performance-considerations)).

:::warning Not a security boundary
Node's `vm` isolates variables, not privileges. Code running here can reach the host process through the objects it is handed, so treat a pre-processing function as code running with EmailEngine's own permissions. Only operators you trust with the server should be able to edit routes.
:::

## Script Environment Variables (scriptEnv)

`scriptEnv` is a runtime setting holding a JSON object of values that every pre-processing function can read as `env`. Use it for API keys, shared secrets and feature flags instead of writing them into the function text, where they would be visible on every route page.

### Configuring scriptEnv

In the admin UI, open **Configuration** > **Webhooks** and fill in the **Script Variables (JSON)** editor. Through the API, `scriptEnv` is a string containing the JSON object (up to 1 MB):

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scriptEnv": "{\"CLASSIFIER_URL\":\"https://classifier.example.com/v1/classify\",\"CLASSIFIER_KEY\":\"sk-abc123\",\"SLACK_WEBHOOK_URL\":\"https://hooks.slack.com/services/T000/B000/XXXX\",\"ENVIRONMENT\":\"production\"}"
  }'
```

The value is parsed on every function execution, so a change applies to the next event without a restart. The whole object is replaced on each update; include every key you want to keep.

Read it back with `GET /v1/settings?scriptEnv=true`.

### Using env in Scripts

```javascript
if (env.ENVIRONMENT === "development") {
  return true;
}

// In production, only INBOX messages
return payload.path === "INBOX";
```

```javascript
// Post a notification to Slack from a map function, then send the original payload on
if (env.SLACK_WEBHOOK_URL) {
  await fetch(env.SLACK_WEBHOOK_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text: `New email from ${payload.data.from.address}` })
  });
}
```

## Examples

### Skip Automated Mail

```javascript
if (payload.data.headers && payload.data.headers["auto-submitted"] && payload.data.headers["auto-submitted"][0] !== "no") {
  return false;
}

const automatedSenders = [/noreply/i, /no-reply/i, /donotreply/i, /mailer-daemon/i, /postmaster/i];
const address = payload.data.from && payload.data.from.address;
if (address && automatedSenders.some(pattern => pattern.test(address))) {
  return false;
}

const automatedSubjects = [/out of office/i, /automatic reply/i, /auto-reply/i, /mail delivery fail/i];
if (payload.data.subject && automatedSubjects.some(pattern => pattern.test(payload.data.subject))) {
  return false;
}

return true;
```

### Route by Sender Domain

```javascript
const address = payload.data.from && payload.data.from.address;
if (!address) {
  return false;
}

const domain = address.split("@")[1].toLowerCase();
return ["important-client.com", "partner.example"].includes(domain);
```

### Extract Reference Numbers

```javascript
if (payload.data.subject) {
  const ticketMatch = payload.data.subject.match(/#(\d+)|TICKET-(\d+)/i);
  if (ticketMatch) {
    payload.ticketId = ticketMatch[1] || ticketMatch[2];
  }

  const orderMatch = payload.data.subject.match(/order\s*#?:?\s*(\d+)/i);
  if (orderMatch) {
    payload.orderId = orderMatch[1];
  }
}

const recipients = [...(payload.data.to || []), ...(payload.data.cc || [])].map(entry => entry.address);
payload.allRecipients = [...new Set(recipients)];

return payload;
```

### Categorise by Subject

```javascript
const categories = {
  billing: ["invoice", "payment", "receipt", "billing"],
  support: ["support", "help", "question", "issue"],
  sales: ["quote", "proposal", "pricing", "demo"]
};

const subject = (payload.data.subject || "").toLowerCase();
payload.category = "general";

for (const [category, keywords] of Object.entries(categories)) {
  if (keywords.some(keyword => subject.includes(keyword))) {
    payload.category = category;
    break;
  }
}

return payload;
```

### Redact Sensitive Content

Text bodies are only present in the payload when the `notifyText` setting is on; see [Webhook content](/docs/webhooks/overview#advanced-webhook-settings).

```javascript
const patterns = {
  ssn: /\b\d{3}-\d{2}-\d{4}\b/g,
  card: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g
};

if (payload.data.text && payload.data.text.plain) {
  const original = payload.data.text.plain;
  payload.data.text.plain = original.replace(patterns.ssn, "[REDACTED]").replace(patterns.card, "[REDACTED]");
  payload.containsSensitiveData = payload.data.text.plain !== original;
}

return payload;
```

### Summarise Attachments

```javascript
if (payload.data.attachments) {
  payload.attachmentStats = {
    total: payload.data.attachments.length,
    images: payload.data.attachments.filter(a => a.contentType && a.contentType.startsWith("image/")).length,
    totalSize: payload.data.attachments.reduce((sum, a) => sum + (a.encodedSize || 0), 0)
  };
}

return payload;
```

## Configuring a Route

1. Open **Integrations** > **Webhook Routes** and create a route, or open an existing one and click **Edit**
2. Enter the filter in the **Filter function** editor. Click **Set test payload** to load one of the example payloads, or paste your own JSON; the **Evaluation result** line updates as you type, showing `filter matches` or `no match`
3. Enter the map in the **Map function** editor. Its preview shows the body the route would deliver for the test payload, and reads `Filter function must return true to activate output mapping` until the filter matches
4. Save. Routes are re-read on the next event, so the change applies without a restart

The route's page shows the saved functions under the **Filter function** and **Map function** tabs, and an **Error log** tab appears once a function has thrown.

![Filter Function Editor](/img/screenshots/webhooks/webhook-route-filter-function.png)

## Performance Considerations

Every enabled route runs its filter for every event EmailEngine raises, on the thread that raised it. Keep filters cheap:

```javascript
// Cheap: property checks
return payload.path === "INBOX" && !(payload.data.headers && payload.data.headers["auto-submitted"]);
```

Calling an external service inside a filter puts that service's latency on the path of every event for every account:

```javascript
// Expensive: a network round trip per event, per route
const response = await fetch(env.CLASSIFIER_URL, {
  method: "POST",
  headers: { Authorization: `Bearer ${env.CLASSIFIER_KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({ subject: payload.data.subject })
});
const result = await response.json();
return result.category === "important";
```

:::tip Let the webhook handler do the slow work
Pass the event through and classify it in your webhook handler, where a slow call delays one delivery instead of every event on the instance. Pre-processing is for decisions that can be made from the payload alone.
:::

Return as early as the answer is known, and compute a value once when several checks need it:

```javascript
if (payload.path !== "INBOX") return false;
if (!payload.data.from || !payload.data.from.address) return false;

const subject = (payload.data.subject || "").toLowerCase();
payload.isUrgent = subject.includes("urgent") || subject.includes("important");
payload.isBilling = subject.includes("invoice") || subject.includes("payment");

return payload;
```

## Debugging

### Using the Logger

```javascript
logger.info({ msg: "Evaluating route", account: payload.account, path: payload.path });

const result = payload.path === "INBOX";
logger.debug({ msg: "Filter decision", result });

return result;
```

Entries appear in EmailEngine's stdout log with `component: "subscript"` and a `subscript` field naming the function, `webhooks:filter:<routeId>` or `webhooks:map:<routeId>`:

```bash
emailengine | jq -c 'select(.component == "subscript")'
```

A thrown error is logged by the webhook subsystem as `Failed to execute webhook script`, with `type` (`filter` or `map`) and `webhook` (the route ID), and is also stored on the route's **Error log** tab together with the payload that triggered it, which is usually the quicker place to look.

### Timing a Function

```javascript
const start = Date.now();

payload.customField = "processed";

logger.info({ msg: "Map function finished", duration: Date.now() - start });
return payload;
```

## HTML Email Pre-Processing for Web Display

Not to be confused with the functions above. EmailEngine can also pre-process message HTML into a form that is safe to inject into a web page, sanitizing the markup, inlining images, and folding quoted thread history into a collapsible block. That is covered on its own page: [Web-Safe HTML](/docs/receiving/web-safe-html).

## See Also

- [Webhook routing](/docs/webhooks/webhook-routing) - Creating routes and the examples that use these functions
- [Webhooks overview](/docs/webhooks/overview) - The events and payloads the functions receive
- [Webhooks API](/docs/api-reference/webhooks-api) - Managing routes and their functions programmatically
- [AI processing](/docs/integrations/ai-chatgpt#ai-pre-processing-filter-openaipreprocessingfn) - The filter that selects messages for LLM processing
- [Web-safe HTML](/docs/receiving/web-safe-html) - The other kind of pre-processing

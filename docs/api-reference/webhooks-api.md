---
title: Webhooks API
description: API endpoints for managing webhooks and receiving real-time email notifications
sidebar_position: 6
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Webhooks API

The Webhooks API configures event notifications from EmailEngine to your application. Instead of polling for changes, EmailEngine posts a JSON payload to your endpoint when an event occurs.

## Overview

The webhook system provides:

- **Event filtering**: an allowlist of event types to deliver
- **Automatic retries**: a failed delivery is retried with exponential backoff
- **Signed payloads**: an HMAC signature header on every request
- **Multiple routes**: additional targets with their own filter and transform functions

```mermaid
graph LR
    EmailEngine[EmailEngine]
    Event[Event occurs]
    POST[HTTP POST]
    App[Your Application]

    EmailEngine --> Event
    Event --> POST
    POST --> App

    style EmailEngine fill:#e1f5ff
    style Event fill:#fff4e1
    style POST fill:#f3e5f5
    style App fill:#e8f5e9
```

### Webhooks vs Polling

| Aspect | Webhooks | Polling |
|--------|----------|---------|
| Latency | As soon as EmailEngine notices the change | Depends on the interval |
| Load on EmailEngine | One request per event | One request per poll, whether or not anything changed |
| Complexity | Needs a public endpoint | Needs a scheduler |
| Missed events | Retried automatically | Anything between two polls has to be reconstructed |

## Webhook Management

### 1. Register Webhook

The default webhook target and the delivery options are ordinary settings, written with `POST /v1/settings`.

**Endpoint:** `POST /v1/settings`

[Detailed API reference](/docs/api/post-v-1-settings)

**Settings:**

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `webhooks` | string | none | Target URL that receives the POST requests. `http` or `https` |
| `webhooksEnabled` | boolean | `false` | Turns webhook delivery on or off for all accounts |
| `webhookEvents` | array | none | Event types to deliver. Use `["*"]` for all events |
| `webhooksCustomHeaders` | array | none | Extra headers on every request, as `{ "key": ..., "value": ... }` objects |
| `notifyHeaders` | array | none | Message headers to include in `messageNew` payloads, for example `["List-ID"]`. `["*"]` includes every header |
| `notifyText` | boolean | `true` | Include the plain text body in `messageNew` payloads |
| `notifyTextSize` | integer | `2097152` | Maximum bytes of text content to include |
| `notifyWebSafeHtml` | boolean | `false` | Include sanitized [web-safe HTML](/docs/receiving/web-safe-html) in payloads |
| `notifyAttachments` | boolean | `false` | Include attachment data in payloads |
| `notifyAttachmentSize` | integer | none | Maximum bytes per attachment to include |
| `notifyCalendarEvents` | boolean | `false` | Include parsed calendar events in payloads |
| `inboxNewOnly` | boolean | `false` | Emit `messageNew` only for messages arriving in the Inbox |

`notifyText` and `notifyTextSize` are seeded on first start; the other values are unset until you write them, which behaves as `false` or none.

:::caution Setting a URL is not enough
Delivery only starts once `webhooksEnabled` is `true` and `webhookEvents` lists the events you want. Setting `webhooks` alone leaves a correctly configured endpoint that never receives anything.
:::

**Example:**

<Tabs groupId="language">
<TabItem value="curl" label="cURL" default>

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhook",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageSent"]
  }'
```

</TabItem>
<TabItem value="nodejs" label="Node.js">

```javascript
const res = await fetch('https://emailengine.example.com/v1/settings', {
  method: 'POST',
  headers: {
    Authorization: 'Bearer YOUR_ACCESS_TOKEN',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    webhooks: 'https://your-app.com/webhook',
    webhooksEnabled: true,
    webhookEvents: ['messageNew', 'messageSent']
  })
});

const { updated } = await res.json();
console.log('Updated settings:', updated);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import requests

response = requests.post(
    'https://emailengine.example.com/v1/settings',
    headers={'Authorization': 'Bearer YOUR_ACCESS_TOKEN'},
    json={
        'webhooks': 'https://your-app.com/webhook',
        'webhooksEnabled': True,
        'webhookEvents': ['messageNew', 'messageSent']
    }
)

print('Updated settings:', response.json()['updated'])
```

</TabItem>
</Tabs>

**Response:**

The response lists the setting keys that were changed:

```json
{
  "updated": ["webhooks", "webhooksEnabled", "webhookEvents"]
}
```

An account can override the target with its own `webhooks` value in the account record, and a webhook route can send a subset of events somewhere else. See [Webhook routing](/docs/webhooks/webhook-routing) for how the three combine.

### 2. List Webhook Routes

**Endpoint:** `GET /v1/webhookRoutes`

[Detailed API reference](/docs/api/get-v-1-webhookroutes)

Takes `page` (zero-indexed) and `pageSize` query parameters.

```bash
curl "https://emailengine.example.com/v1/webhookRoutes" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**

```json
{
  "total": 1,
  "page": 0,
  "pages": 1,
  "webhooks": [
    {
      "id": "AAABgS-UcAYAAAABAA",
      "name": "Send to Slack",
      "description": "Notify Slack on new messages",
      "targetUrl": "https://your-app.com/webhook",
      "enabled": true,
      "created": "2021-02-17T13:43:18.860Z",
      "updated": "2021-02-17T13:45:00.000Z",
      "tcount": 123,
      "webhookErrorFlag": null,
      "customHeaders": []
    }
  ]
}
```

`tcount` counts how many times the route has been applied. `webhookErrorFlag` carries the `message` of the last failed delivery, or `null`.

### 3. Get Webhook Route

**Endpoint:** `GET /v1/webhookRoutes/webhookRoute/{webhookRoute}`

[Detailed API reference](/docs/api/get-v-1-webhookroutes-webhookroute-webhookroute)

```bash
curl "https://emailengine.example.com/v1/webhookRoutes/webhookRoute/AAABgS-UcAYAAAABAA" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**

```json
{
  "id": "AAABgS-UcAYAAAABAA",
  "name": "Send to Slack",
  "description": "Notify Slack on new messages",
  "targetUrl": "https://your-app.com/webhook",
  "enabled": true,
  "created": "2021-02-17T13:43:18.860Z",
  "updated": "2021-02-17T13:45:00.000Z",
  "tcount": 123,
  "v": 1,
  "webhookErrorFlag": null,
  "customHeaders": [],
  "content": {
    "fn": "return true;",
    "map": "payload.ts = Date.now(); return payload;"
  }
}
```

`content.fn` is the filter function and `content.map` the mapping function, both as source text; either is `null` when not set.

:::note Webhook routes are read-only through the API
The API exposes only the two read endpoints above. There are no endpoints to create, update, or delete a webhook route: manage them in the admin interface under **Webhook Routes**.

This is separate from the default webhook target, which you configure with `POST /v1/settings` as shown above. Routes fan events out to additional destinations with their own filter and map functions.
:::

## Webhook Configuration

### Target URL

The webhook endpoint must:

- Be reachable from the EmailEngine host. By default, link-local addresses are blocked and redirects are not followed; see [`EENGINE_WEBHOOK_EGRESS_POLICY`](/docs/configuration/environment-variables#webhook-delivery)
- Respond within the delivery timeout, 30 seconds by default (`EENGINE_WEBHOOK_TIMEOUT`)
- Return a `2xx` status code for success. Anything else, or a timeout, counts as a failed attempt. A redirect is refused rather than followed, and is not retried (since v2.75.0)

### Event Filters

`webhookEvents` is an allowlist. Leaving it out or setting it to `[]` delivers nothing at all:

```json
{
  "webhooks": "https://your-app.com/webhook",
  "webhookEvents": ["messageNew", "messageDeleted", "messageSent", "messageDeliveryError"]
}
```

Subscribe to every event, including types added in later EmailEngine releases, with the `*` wildcard:

```json
{
  "webhooks": "https://your-app.com/webhook",
  "webhookEvents": ["*"]
}
```

### Custom Headers

`webhooksCustomHeaders` adds headers to every request to the default target. Each entry is a `key` and `value` pair:

```json
{
  "webhooks": "https://your-app.com/webhook",
  "webhooksCustomHeaders": [
    { "key": "X-API-Key", "value": "your-secret-key" },
    { "key": "X-Source", "value": "emailengine" }
  ]
}
```

### Authentication

**Bearer token**, as a custom header:

```json
{
  "webhooks": "https://your-app.com/webhook",
  "webhooksCustomHeaders": [
    { "key": "Authorization", "value": "Bearer YOUR_SECRET_TOKEN" }
  ]
}
```

**Basic auth**, as credentials in the URL. EmailEngine strips them from the URL and sends them as an `Authorization: Basic` header; they are redacted in logs:

```json
{
  "webhooks": "https://user:password@your-app.com/webhook"
}
```

Neither replaces the [signature](#webhook-signatures), which is what proves the payload came from your EmailEngine instance and was not altered.

## Webhook Payload

### Common Payload Structure

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-01-15T10:30:00.000Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageNew",
  "data": {}
}
```

| Field | Type | Description |
|-------|------|-------------|
| `serviceUrl` | string | The configured `serviceUrl` of the EmailEngine instance that sent the event |
| `account` | string | Account identifier |
| `date` | string | ISO 8601 timestamp of when the event was generated |
| `path` | string | Folder the event relates to. Omitted for events that are not about a folder |
| `specialUse` | string | Special-use flag of that folder, such as `\Inbox` or `\Sent`. Only present when the folder has one |
| `event` | string | Event type, for example `messageNew` |
| `data` | object | Event-specific payload |

`path` and `specialUse` are only present for folder-scoped events, so an account-level event such as `authenticationError` carries neither. The event ID travels in the `X-EE-Wh-Event-Id` header rather than in the body.

### Event-Specific Fields

Each event type carries its own `data` object. See the [Webhook Events Reference](/docs/reference/webhook-events) for every field, and the per-event pages linked from it for examples.

**Example, `messageNew` with the default text settings:**

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-01-15T10:30:00.000Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageNew",
  "data": {
    "id": "AAAABAABNc",
    "uid": 12345,
    "date": "2025-01-15T10:29:58.000Z",
    "flags": [],
    "unseen": true,
    "size": 4096,
    "subject": "New Email",
    "from": {
      "name": "John Doe",
      "address": "john@example.com"
    },
    "to": [
      { "address": "user@example.com" }
    ],
    "messageId": "<abc123@example.com>",
    "text": {
      "id": "AAAADAAABy6TkaMxLjGRozEuMpA",
      "encodedSize": {
        "plain": 5
      },
      "plain": "Hello"
    }
  }
}
```

### Request Headers

EmailEngine sets these headers on every delivery:

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `User-Agent` | `emailengine-app/<version> (+https://emailengine.app/)` |
| `X-EE-Wh-Id` | Queue job ID of this delivery |
| `X-EE-Wh-Attempts-Made` | Number of attempts made before this one, so `0` on the first |
| `X-EE-Wh-Queued-Time` | Time since the event was queued, in whole seconds, for example `5s` |
| `X-EE-Wh-Event-Id` | Event ID, when the event has one. Stable across retries of the same delivery |
| `X-EE-Wh-Custom-Route` | ID of the webhook route, when the delivery was produced by one |
| `X-EE-Wh-Signature` | HMAC-SHA256 of the body, see [Webhook Signatures](#webhook-signatures) |

## Event Types

`webhookEvents` accepts the following names. Payloads are documented in the [Webhook Events Reference](/docs/reference/webhook-events).

### Account Events

| Event | Trigger |
|-------|---------|
| `accountAdded` | Account registered |
| `accountInitialized` | The initial mailbox synchronization finished |
| `accountDeleted` | Account deleted |
| `authenticationSuccess` | The mail server accepted the credentials |
| `authenticationError` | The mail server rejected the credentials |
| `connectError` | The mail server could not be reached |

### Message Events

| Event | Trigger |
|-------|---------|
| `messageNew` | New message detected in a folder |
| `messageUpdated` | Flags or labels of a message changed |
| `messageDeleted` | Message removed from a folder |
| `messageMissing` | A message that should exist was not found, indicating a synchronization error |

### Mailbox Events

| Event | Trigger |
|-------|---------|
| `mailboxNew` | Folder created |
| `mailboxDeleted` | Folder deleted |
| `mailboxReset` | EmailEngine rebuilt the folder's index, invalidating previously tracked UIDs |

### Sending Events

| Event | Trigger |
|-------|---------|
| `messageSent` | The receiving server accepted a queued message |
| `messageDeliveryError` | A delivery attempt failed and will be retried |
| `messageFailed` | Every delivery attempt failed |
| `messageBounce` | A bounce notification arrived for a sent message |
| `messageComplaint` | A feedback loop complaint (ARF) arrived |

### Tracking and List Events

| Event | Trigger |
|-------|---------|
| `trackOpen` | A tracked message was opened |
| `trackClick` | A tracked link was clicked |
| `listSubscribe` | A recipient subscribed to a mail merge list |
| `listUnsubscribe` | A recipient unsubscribed from a mail merge list |

### Export Events

| Event | Trigger |
|-------|---------|
| `exportCompleted` | A bulk message export finished |
| `exportFailed` | A bulk message export failed |

## Security

### Webhook Signatures

Every delivery carries a signature header:

```text
X-EE-Wh-Signature: <base64url-encoded HMAC-SHA256>
```

The signature is HMAC-SHA256 over the **raw** request body, keyed with the `serviceSecret` setting and base64url encoded. `serviceSecret` is generated when EmailEngine first starts; read it under **Configuration** > **Security** > **Service Secret**, or set your own value there or through `POST /v1/settings`.

To verify, compute the same HMAC over the bytes you received and compare in constant time. Parsing the JSON first and re-serializing it will not reproduce the same bytes, so the signature will not match.

<Tabs groupId="language">
<TabItem value="nodejs" label="Node.js" default>

```javascript
const crypto = require('crypto');
const express = require('express');

const app = express();

function verifyWebhook(rawBody, signature, secret) {
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('base64url');

  const a = Buffer.from(expected);
  const b = Buffer.from(signature || '', 'utf8');

  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

// express.raw keeps the exact bytes that were signed
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-ee-wh-signature'];

  if (!verifyWebhook(req.body, signature, process.env.WEBHOOK_SECRET)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const event = JSON.parse(req.body.toString());
  console.log('Event:', event.event);

  res.json({ success: true });
});

app.listen(8080);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import base64
import hashlib
import hmac
import os

from flask import Flask, jsonify, request

app = Flask(__name__)


def verify_webhook(payload, signature, secret):
    computed = hmac.new(secret.encode(), payload, hashlib.sha256).digest()
    expected = base64.urlsafe_b64encode(computed).rstrip(b'=').decode()
    return hmac.compare_digest(signature or '', expected)


@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-EE-Wh-Signature')
    payload = request.get_data()
    # The secret is the serviceSecret setting from EmailEngine
    secret = os.environ['WEBHOOK_SECRET']

    if not verify_webhook(payload, signature, secret):
        return jsonify({'error': 'Invalid signature'}), 401

    event = request.get_json()
    print(f"Event: {event['event']}")
    return jsonify({'success': True})
```

</TabItem>
</Tabs>

### Source Address Filtering

Restricting the endpoint to the EmailEngine host's address is a useful second layer:

```nginx
location /webhook {
    allow 203.0.113.10;
    deny all;
    proxy_pass http://127.0.0.1:8080;
}
```

It is not a substitute for verifying the signature: an allowlist only proves where a request came from, not that its contents are untampered. If EmailEngine sits behind a proxy or NAT, the address your application sees is the proxy's, so verify the signature there too.

### HTTPS

Use `https` targets outside development. The payload carries message content, and a plain `http` target exposes it, and the custom headers, to anything on the path.

## Testing Webhooks

### Send a Test Webhook

The admin interface can post a sample event to the configured target:

1. Open **Configuration** > **Webhooks**
2. Click **Send test webhook**
3. Check that your endpoint received it

### Testing Tools

**Webhook.site** gives you a URL that displays everything posted to it, with no code:

```text
https://webhook.site/
```

**ngrok** exposes a local receiver to the internet:

```bash
ngrok http 8080
```

Use the generated `https://` URL, with `/webhook` appended, as the target.

### Local Testing

A throwaway receiver that prints what arrives:

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook', (req, res) => {
  console.log(req.body.event, req.body.account);
  console.dir(req.body.data, { depth: null });
  res.json({ success: true });
});

app.listen(8080, () => console.log('Listening on 8080'));
```

This one skips signature verification on purpose, which is fine while you are inspecting payloads locally but not something to carry into production. See [Webhook Signatures](#webhook-signatures) for the verified version.

**Post a sample payload to it:**

```bash
curl -X POST http://localhost:8080/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "event": "messageNew",
    "account": "test@example.com",
    "data": {
      "subject": "Test"
    }
  }'
```

### Debugging Tips

**Check the delivery log:**

```bash
docker logs emailengine | grep webhook
```

**Check that the endpoint is reachable from the EmailEngine host:**

```bash
curl -X POST https://your-app.com/webhook \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

**Common causes of failed deliveries:**

- Endpoint not reachable from the EmailEngine host (firewall, DNS, or a destination blocked by the egress policy)
- Invalid HTTPS certificate
- No response within the timeout (30 seconds by default)
- Non-`2xx` status code
- Signature verification failing on your side because the body was re-serialized before hashing

Failed deliveries are retried up to 10 times with exponential backoff. The **Webhooks queue** on the dashboard shows what is waiting and what has failed; see [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Building a Receiver

A production webhook endpoint has four jobs, in this order:

1. **Verify the signature** over the raw body, before parsing anything.
2. **Deduplicate** on the `X-EE-Wh-Event-Id` header, because a retried delivery repeats an event you may already have handled.
3. **Acknowledge with `2xx`** as soon as the payload is durably stored.
4. **Do the work afterwards**, off the request path, so a slow database never turns into a retry.

Getting the order wrong is what causes the two failure modes seen in practice: parsing before verifying makes the signature unverifiable, and working before acknowledging makes EmailEngine retry a request you are still processing.

```javascript
const crypto = require('crypto');
const express = require('express');

const app = express();
const SECRET = process.env.WEBHOOK_SECRET; // serviceSecret from EmailEngine settings

// Raw body: the signature covers the exact bytes, not a re-serialized object
app.post('/webhook', express.raw({ type: 'application/json' }), async (req, res) => {
  // 1. Verify before trusting anything in the payload
  const expected = crypto.createHmac('sha256', SECRET).update(req.body).digest('base64url');
  const given = Buffer.from(req.headers['x-ee-wh-signature'] || '', 'utf8');
  const want = Buffer.from(expected);

  if (given.length !== want.length || !crypto.timingSafeEqual(want, given)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const eventId = req.headers['x-ee-wh-event-id'];
  const event = JSON.parse(req.body.toString());

  // 2 + 3. Record it, then acknowledge. A conflict means we already have it
  const isNew = await store.insertIfAbsent(eventId, event);
  res.json({ success: true });

  if (!isNew) return;

  // 4. Work happens after the response
  queue.push(event).catch(err => console.error('Enqueue failed', eventId, err));
});

app.listen(8080);
```

Dedupe on the event ID rather than on a composite key you build yourself. `X-EE-Wh-Event-Id` is stable across retries of the same delivery, whereas a key assembled from event type and message ID collides for legitimately distinct events, such as a message that is flagged twice.

:::caution An in-memory set is not deduplication
A `Set` in the process loses everything on restart and is not shared between instances, so a redeploy or a second replica reprocesses events. Use whatever your application already treats as durable, with a unique constraint on the event ID.
:::

Handle events by type once the payload is on your queue. Only `event` is guaranteed on every payload, so branch on it and ignore what you do not consume. New event types are added in EmailEngine releases, and a receiver that throws on an unrecognized type starts failing after an upgrade.

## See Also

- [Webhooks overview](/docs/webhooks/overview) - Setup, delivery, retries, and debugging
- [Webhook routing](/docs/webhooks/webhook-routing) - Per-account and per-event targets
- [Webhook events reference](/docs/reference/webhook-events) - Every event and its payload
- [Pre-processing functions](/docs/advanced/pre-processing) - Filtering a payload before it is sent
- [Environment variables](/docs/configuration/environment-variables#webhook-delivery) - Delivery timeout and egress policy

---
title: Webhooks API
description: API endpoints for managing webhooks and receiving real-time email notifications
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Webhooks API

The Webhooks API allows you to configure real-time event notifications from EmailEngine to your application. Instead of polling for changes, webhooks push notifications to your endpoint when events occur.

## Overview

### Webhook System Architecture

EmailEngine's webhook system provides:

- **Real-time notifications**: Instant event delivery
- **Event filtering**: Subscribe to specific event types
- **Automatic retries**: Failed deliveries are retried with exponential backoff
- **Signature verification**: Secure webhook payload authentication
- **Multiple routes**: Configure different endpoints for different accounts

### Event-Driven Integration

Webhooks enable event-driven architecture:

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

Benefits:
- No polling overhead
- Instant notification
- Scalable to high-volume accounts
- Reduced API calls

### Webhooks vs Polling

| Aspect | Webhooks | Polling |
|--------|----------|---------|
| Latency | Instant | Depends on interval |
| Efficiency | High (push) | Low (pull) |
| Server Load | Low | High |
| Complexity | Medium | Low |
| Reliability | Auto-retry | Manual |

## Webhook Management

### 1. Register Webhook

Configure a webhook endpoint to receive events.

**Endpoint:** `POST /v1/settings`

[Detailed API reference →](/docs/api/post-v-1-settings)

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `webhooks` | string | Yes | Webhook endpoint URL |
| `webhooksEnabled` | boolean | No | Turns webhook delivery on or off for all accounts |
| `webhookEvents` | array | No | Event types to receive. Use `["*"]` for all events |

:::caution Setting a URL is not enough
Delivery only starts once `webhooksEnabled` is `true`. Setting `webhooks` alone leaves a correctly configured endpoint that never receives anything.
:::

**Example:**

<Tabs groupId="language">
<TabItem value="curl" label="cURL" default>

```bash
curl -X POST http://localhost:3000/v1/settings \
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
const res = await fetch('http://localhost:3000/v1/settings', {
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
response = requests.post(
    'http://localhost:3000/v1/settings',
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

### 2. List Webhook Routes

Retrieve all configured webhook routes.

**Endpoint:** `GET /v1/webhookRoutes`

[Detailed API reference →](/docs/api/get-v-1-webhookroutes)

**Example:**

```bash
curl "http://localhost:3000/v1/webhookRoutes" \
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
      "tcount": 123
    }
  ]
}
```

### 3. Get Webhook Route

Retrieve details of a specific webhook route.

**Endpoint:** `GET /v1/webhookRoutes/webhookRoute/:webhookRoute`

[Detailed API reference →](/docs/api/get-v-1-webhookroutes-webhookroute-webhookroute)

**Example:**

```bash
curl "http://localhost:3000/v1/webhookRoutes/webhookRoute/AAABgS-UcAYAAAABAA" \
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
  "content": {
    "fn": "return true;",
    "map": "payload.ts = Date.now(); return payload;"
  }
}
```

:::note Webhook routes are read-only through the API
The API exposes only the two read endpoints above (`GET /v1/webhookRoutes` and `GET /v1/webhookRoutes/webhookRoute/:webhookRoute`). There are no API endpoints to create, update, or delete a webhook route - manage them in the EmailEngine dashboard under **Webhooks**.

This is separate from the primary webhook target, which you configure with `POST /v1/settings` (see "Register Webhook" above). Webhook routes are an advanced feature for fanning events out to multiple destinations with custom filter/map functions.
:::

## Webhook Configuration

### Target URL

The webhook endpoint must:
- Use HTTPS (HTTP only for development)
- Be publicly accessible
- Respond within 30 seconds
- Return 2xx status code for success

**Valid URLs:**
```
https://your-app.com/webhook
https://api.yourservice.com/emailengine/events
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX
```

### Event Filters

Subscribe to specific event types:

```javascript
{
  webhooks: 'https://your-app.com/webhook',
  webhookEvents: [
    'messageNew',        // New message received
    'messageDeleted',    // Message deleted
    'messageSent',       // Message sent successfully
    'messageDeliveryError' // Send failed
  ]
}
```

**Subscribe to all events** with the `*` wildcard, including event types added in later EmailEngine releases:

```javascript
{
  webhooks: 'https://your-app.com/webhook',
  webhookEvents: ['*']
}
```

:::caution An empty list means no events, not all events
`webhookEvents` is an allowlist. Leaving it out or setting it to `[]` delivers nothing at all. To receive everything, set it to `['*']` explicitly.
:::

### Custom Headers

Add custom headers to webhook requests using an array of key-value objects:

```javascript
{
  webhooks: 'https://your-app.com/webhook',
  webhooksCustomHeaders: [
    { key: 'X-API-Key', value: 'your-secret-key' },
    { key: 'X-Source', value: 'emailengine' }
  ]
}
```

### Authentication

**Bearer Token:**
```javascript
{
  webhooks: 'https://your-app.com/webhook',
  webhooksCustomHeaders: [
    { key: 'Authorization', value: 'Bearer YOUR_SECRET_TOKEN' }
  ]
}
```

**Basic Auth:**
Include credentials in URL (not recommended for production):
```javascript
{
  webhooks: 'https://user:password@your-app.com/webhook'
}
```

## Webhook Payload

### Common Payload Structure

All webhooks follow this structure:

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

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `serviceUrl` | string | The configured service URL of the EmailEngine instance that sent the event |
| `account` | string | Account identifier |
| `date` | string | ISO timestamp of when the event was generated |
| `path` | string | Folder the event relates to. Omitted for events that are not about a folder |
| `specialUse` | string | Special-use flag of that folder, such as `\Inbox` or `\Sent`. Only present when the folder has one |
| `event` | string | Event type, for example `messageNew` |
| `data` | object | Event-specific payload |

`path` and `specialUse` are only present for folder-scoped events, so an account-level event such as `authenticationError` carries neither. The event ID travels in the `X-EE-Wh-Event-Id` header rather than in the body.

### Event-Specific Fields

Each event type includes specific data in the `data` field. See [Webhook Events Reference](/docs/reference/webhook-events) for complete details.

**Example - messageNew:**
```json
{
  "event": "messageNew",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "id": "AAAABAABNc",
    "uid": 12345,
    "path": "INBOX",
    "subject": "New Email",
    "from": {
      "name": "John Doe",
      "address": "john@example.com"
    },
    "date": "2025-01-15T10:30:00.000Z",
    "unseen": true
  }
}
```

### Retry Metadata

EmailEngine includes retry information in HTTP headers (not in the payload):

| Header | Description |
|--------|-------------|
| `X-EE-Wh-Id` | Unique webhook job ID |
| `X-EE-Wh-Attempts-Made` | Number of delivery attempts made |
| `X-EE-Wh-Queued-Time` | Time since webhook was queued (e.g., "5s") |
| `X-EE-Wh-Event-Id` | Event ID (if available) |

## Event Types Overview

Complete list of webhook events (see [Webhook Events Reference](/docs/reference/webhook-events) for detailed payloads):

### Account Events

| Event | Trigger |
|-------|---------|
| `accountAdded` | Account registered |
| `accountDeleted` | Account deleted |
| `authenticationError` | Authentication failed |
| `connectError` | Connection to server failed |
| `accountInitialized` | Account first connected |

### Message Events

| Event | Trigger |
|-------|---------|
| `messageNew` | New message received |
| `messageDeleted` | Message deleted from mailbox |
| `messageUpdated` | Message flags changed |
| `messageMissing` | Message disappeared from mailbox |

### Mailbox Events

| Event | Trigger |
|-------|---------|
| `mailboxNew` | New folder created |
| `mailboxDeleted` | Folder deleted |
| `mailboxReset` | Mailbox sync was reset |

### Sending Events

| Event | Trigger |
|-------|---------|
| `messageSent` | Message sent successfully |
| `messageDeliveryError` | Sending failed |
| `messageBounce` | Bounce notification received |

## Security

### Webhook Signatures

EmailEngine signs webhook payloads for verification.

**Signature Header:**
```
X-EE-Wh-Signature: <base64url-encoded-hmac>
```

**Verify Signature:**

Compute HMAC-SHA256 over the **raw** request body with the `serviceSecret` as the key, base64url encode it, and compare in constant time. Parsing the JSON first and re-serializing it will not reproduce the same bytes, so the signature will not match.

<Tabs groupId="language">
<TabItem value="nodejs" label="Node.js" default>

```javascript
const crypto = require('crypto');
const express = require('express');

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
```

</TabItem>
<TabItem value="python" label="Python">

```python
import hmac
import hashlib
import base64

def verify_webhook(payload, signature, secret):
    # Compute HMAC-SHA256 and encode as base64url (no padding)
    computed = hmac.new(
        secret.encode(),
        payload,  # raw bytes
        hashlib.sha256
    ).digest()
    expected = base64.urlsafe_b64encode(computed).rstrip(b'=').decode()

    return hmac.compare_digest(signature, expected)

# Flask example
@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-EE-Wh-Signature')
    payload = request.get_data()  # raw bytes
    # Note: The secret is the serviceSecret from EmailEngine settings
    secret = os.environ['WEBHOOK_SECRET']

    if not verify_webhook(payload, signature, secret):
        return jsonify({'error': 'Invalid signature'}), 401

    event = request.json
    print(f"Event: {event['event']}")
    return jsonify({'success': True})
```

</TabItem>
</Tabs>

### IP Whitelisting

Restrict webhook access to EmailEngine's IP:

**Nginx:**
```nginx
location /webhook {
    allow 1.2.3.4;  # EmailEngine IP
    deny all;
    proxy_pass http://localhost:3000;
}
```

Filtering by source address is a useful second layer, but it is not a substitute for verifying the signature: an allowlist only proves where a request came from, not that its contents are untampered. If EmailEngine sits behind a proxy or NAT, the address your application sees is the proxy's, so verify the signature there too.

### HTTPS Requirement

Production webhooks should use HTTPS:

- Prevents man-in-the-middle attacks
- Encrypts sensitive payload data
- Required for PCI compliance

**Development exception:**
HTTP allowed for localhost testing only.

## Testing Webhooks

### Webhook Tailing Feature

Monitor webhooks in real-time via EmailEngine UI:

1. Navigate to Configuration > Webhooks
2. Click "Tail Webhooks"
3. See live webhook deliveries

### Testing Tools

**RequestBin:**
Create temporary webhook endpoint:
```
https://requestbin.com/
```

**Webhook.site:**
Instant webhook URL for testing:
```
https://webhook.site/
```

**ngrok:**
Expose local server to internet:
```bash
ngrok http 3000
# Use generated URL: https://abc123.ngrok.io/webhook
```

### Local Testing

**A throwaway receiver that prints what arrives:**

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook', (req, res) => {
  console.log(req.body.event, req.body.account);
  console.dir(req.body.data, { depth: null });
  res.json({ success: true });
});

app.listen(3000, () => console.log('Listening on 3000'));
```

This one skips signature verification on purpose, which is fine while you are inspecting payloads locally but not something to carry into production. See [Webhook Signatures](#webhook-signatures) for the verified version.

**Test with curl:**
```bash
curl -X POST http://localhost:3000/webhook \
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

**Check webhook logs in EmailEngine:**
```bash
# View logs with webhook activity
docker logs emailengine | grep webhook
```

**Validate endpoint:**
```bash
# Test your endpoint is accessible
curl -X POST https://your-app.com/webhook \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

**Common issues:**
- Endpoint not accessible (firewall, DNS)
- HTTPS certificate invalid
- Timeout (response > 30s)
- Wrong status code (not 2xx)
- Signature verification failing

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

app.listen(3000);
```

Dedupe on the event ID rather than on a composite key you build yourself. `X-EE-Wh-Event-Id` is stable across retries of the same delivery, whereas a key assembled from event type and message ID collides for legitimately distinct events, such as a message that is flagged twice.

:::caution An in-memory set is not deduplication
A `Set` in the process loses everything on restart and is not shared between instances, so a redeploy or a second replica reprocesses events. Use whatever your application already treats as durable, with a unique constraint on the event ID.
:::

Handle events by type once the payload is on your queue. Only `event` is guaranteed on every payload, so branch on it and ignore what you do not consume. New event types are added in EmailEngine releases, and a receiver that throws on an unrecognized type starts failing after an upgrade.

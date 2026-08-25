---
title: Blocklist Management
sidebar_position: 8
description: Manage email blocklists for suppression lists, bounce handling, and one-click unsubscribe compliance
keywords:
  - blocklist
  - suppression list
  - unsubscribe
  - email list hygiene
  - bounce management
  - mail merge
---

# Blocklist Management

EmailEngine provides blocklist functionality for managing email suppression lists. Blocklists prevent emails from being sent to addresses that have unsubscribed, bounced, or been manually blocked. They integrate with mail merge, one-click unsubscribe (RFC 8058), and bounce detection.

## Overview

Blocklists are collections of email addresses associated with a named list. When sending mail merge campaigns with a `listId`, EmailEngine automatically checks each recipient against the corresponding blocklist and skips blocked addresses.

**Key features:**

- Ad-hoc list creation (lists are created automatically when the first entry is added)
- RFC 8058 one-click unsubscribe support with `List-Unsubscribe` headers
- Per-recipient tracking with source, reason, and timestamp metadata
- Integration with mail merge for automatic recipient filtering
- Webhook notifications for subscribe/unsubscribe events

## How Blocklists Work

A blocklist is only consulted when you send. Passing a `listId` on a mail merge makes EmailEngine check every recipient against the list of that name, skip the ones it finds, and attach one-click unsubscribe headers to the rest so recipients can add themselves to it.

That sending side, including the hosted unsubscribe page and the `serviceUrl` setting it depends on, is covered in [Virtual Mailing Lists](/docs/advanced/virtual-lists). This page covers the store itself: reading it, and writing to it from your own application.

### List ID Format

List IDs must use a subdomain/hostname format:

- **Valid:** `newsletter`, `weekly-updates`, `campaign-2024`, `promo-emails`
- **Invalid:** `my_list` (underscores), `My List` (spaces), `list@domain` (@ symbol)

Lists are created automatically when the first entry is added, so no pre-registration is needed. Removing the last entry deletes the list again.

## API Operations

### List All Blocklists

```bash
curl "https://emailengine.example.com/v1/blocklists" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:

```json
{
  "total": 3,
  "page": 0,
  "pages": 1,
  "blocklists": [
    {"listId": "weekly-newsletter", "count": 42},
    {"listId": "product-updates", "count": 15},
    {"listId": "bounce-hard", "count": 8}
  ]
}
```

[API reference -->](/docs/api/get-v-1-blocklists)

### List Entries in a Blocklist

```bash
curl "https://emailengine.example.com/v1/blocklist/weekly-newsletter?pageSize=50" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:

```json
{
  "listId": "weekly-newsletter",
  "total": 42,
  "page": 0,
  "pages": 1,
  "addresses": [
    {
      "recipient": "bob@example.com",
      "account": "user123",
      "source": "one-click",
      "reason": "unsubscribe",
      "messageId": "<abc@example.com>",
      "created": "2024-10-13T12:10:40.980Z"
    }
  ]
}
```

Each entry includes:

| Field | Description |
|-------|-------------|
| `recipient` | Blocked email address (stored lowercase) |
| `account` | Account that triggered the block |
| `source` | How the entry was added: `one-click` (mail client unsubscribe button), `form` (hosted unsubscribe page), `api` (Blocklists API) or `admin` (admin interface) |
| `reason` | Why the address was blocked: `unsubscribe`, `block`, or custom |
| `messageId` | Original message ID (for unsubscribe entries) |
| `created` | Timestamp when the entry was added |

[API reference -->](/docs/api/get-v-1-blocklist-listid)

### Add an Address to a Blocklist

```bash
curl -X POST "https://emailengine.example.com/v1/blocklist/weekly-newsletter" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "recipient": "spam-reporter@example.com",
    "reason": "complained"
  }'
```

Response:

```json
{
  "success": true,
  "added": true
}
```

The `added` field is `false` if the address was already in the blocklist.

[API reference -->](/docs/api/post-v-1-blocklist-listid)

### Remove an Address from a Blocklist

```bash
curl -X DELETE "https://emailengine.example.com/v1/blocklist/weekly-newsletter?recipient=bob@example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:

```json
{
  "deleted": true
}
```

[API reference -->](/docs/api/delete-v-1-blocklist-listid)

## Bounce-Based Blocking

You can integrate blocklists with bounce detection to automatically suppress addresses that hard bounce. When a `messageBounce` webhook indicates a permanent failure, add the recipient to a blocklist:

```javascript
app.post('/webhooks/emailengine', (req, res) => {
  const event = req.body;
  res.json({ success: true });

  if (event.event === 'messageBounce') {
    const { recipient, action, response } = event.data;
    const recommendedAction = response?.recommendedAction;

    // Add hard bounces to blocklist
    if (action === 'failed' || recommendedAction === 'remove') {
      fetch('https://emailengine.example.com/v1/blocklist/bounce-hard', {
        method: 'POST',
        headers: {
          'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          account: event.account,
          recipient: recipient,
          reason: 'hard-bounce'
        })
      });
    }
  }
});
```

Then reference the `bounce-hard` list in your mail merge campaigns to automatically skip addresses that have bounced:

```json
{
  "listId": "bounce-hard",
  "mailMerge": [{ "to": { "address": "recipient@example.com" } }]
}
```

:::tip Multiple Blocklists
Each mail merge can only reference one `listId`. If you need to check against multiple suppression lists (e.g., both unsubscribes and hard bounces), consolidate them into a single list, or implement pre-send checking in your application by querying each blocklist via the API.
:::

## Webhook Events

Blocklist changes trigger two webhook events:

### listUnsubscribe

Triggered when an address is added to a blocklist via the one-click unsubscribe mechanism.

```json
{
  "event": "listUnsubscribe",
  "account": "user123",
  "data": {
    "recipient": "bob@example.com",
    "messageId": "<abc@example.com>",
    "listId": "weekly-newsletter",
    "remoteAddress": "192.168.1.1",
    "userAgent": "Mozilla/5.0..."
  }
}
```

[See listUnsubscribe reference -->](/docs/webhooks/listunsubscribe)

### listSubscribe

Triggered when a recipient re-subscribes through the hosted unsubscribe page. Removing an address via the DELETE API does not trigger this event.

[See listSubscribe reference -->](/docs/webhooks/listsubscribe)

## Best Practices

1. **Use descriptive list IDs** - Name lists after their purpose: `weekly-newsletter`, `product-announcements`, `transactional-bounces`

2. **Always include unsubscribe links** - Use `{{rcpt.unsubscribeUrl}}` in mail merge templates to provide RFC 8058 compliant unsubscribe links

3. **Handle bounce webhooks** - Automatically add hard-bounced addresses to a blocklist to maintain list hygiene

4. **Monitor blocklist growth** - Regularly review blocklist sizes via the API to track unsubscribe rates

5. **Configure serviceUrl** - Ensure the `serviceUrl` setting is configured so unsubscribe links work correctly

## See Also

- [Virtual Mailing Lists](/docs/advanced/virtual-lists) - The sending side: unsubscribe headers, hosted page and suppression at send time
- [Mail Merge](/docs/sending/mail-merge) - Sending personalized bulk emails
- [Bounce Detection](/docs/advanced/bounces) - Automatic bounce handling
- [Webhook Events Reference](/docs/reference/webhook-events) - All webhook event types

---
title: "listUnsubscribe"
sidebar_position: 23
description: "Webhook event triggered when a recipient unsubscribes from a mailing list via one-click unsubscribe"
---

# listUnsubscribe

The `listUnsubscribe` webhook event is triggered when a recipient unsubscribes from a mailing list through the unsubscribe mechanism EmailEngine adds to messages sent with a `listId`. Use it to keep your own subscriber records in step with the suppression list EmailEngine maintains.

## When This Event is Triggered

The `listUnsubscribe` event fires when a recipient is added to the suppression list of a `listId` through either of the two public endpoints EmailEngine links from the message:

- **One-click unsubscribe (RFC 8058)**: the recipient's mail client sends a `POST` to the signed `List-Unsubscribe` URL. Gmail, Yahoo and other clients do this when the user presses their "Unsubscribe" button
- **The unsubscribe page**: the recipient opens the same URL in a browser and confirms on the form EmailEngine renders

In both cases the request has to carry a valid signature, the account has to exist, and the address must not already be on the suppression list for that `listId`. A repeat request is acknowledged but does not fire the event again.

Adding an address through the [Blocklists API](/docs/api/post-v-1-blocklist-listid) or the admin interface does not fire it; the event reports recipient actions.

## Common Use Cases

- **Mailing list management** - Automatically remove recipients from your mailing lists
- **Subscription database updates** - Keep your subscriber database synchronized with unsubscribe requests
- **Compliance tracking** - Maintain records of unsubscribe requests for regulatory compliance (CAN-SPAM, GDPR)
- **Preference center updates** - Update user preferences in your application
- **Email deliverability** - Honor unsubscribe requests to maintain sender reputation
- **Analytics** - Track unsubscribe rates across different campaigns or list segments

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID that sent the original message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `listUnsubscribe` |
| `data` | object | Yes | Unsubscribe details (see below) |

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `recipient` | string | Yes | Email address that was unsubscribed |
| `messageId` | string | No | Message-ID of the message whose unsubscribe link was used. Absent when the link was generated from the admin interface rather than embedded in a message |
| `listId` | string | Yes | The list the recipient was removed from |
| `remoteAddress` | string | Yes | IP address the request came from. For a one-click unsubscribe this is the mail provider's server, not the recipient |
| `userAgent` | string | No | `User-Agent` header of the request, when one was sent |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "marketing",
  "date": "2025-10-17T14:32:15.882Z",
  "event": "listUnsubscribe",
  "data": {
    "recipient": "customer@company.com",
    "messageId": "<newsletter-2025-10-001@marketing.example.com>",
    "listId": "weekly-newsletter",
    "remoteAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36"
  }
}
```

## Example with Minimal Data

A one-click request from a mail provider that sends no `User-Agent` header:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "notifications",
  "date": "2025-10-17T09:15:22.000Z",
  "event": "listUnsubscribe",
  "data": {
    "recipient": "user@example.org",
    "messageId": "<alert-12345@notifications.example.com>",
    "listId": "product-alerts",
    "remoteAddress": "10.0.0.50"
  }
}
```

## Getting the Unsubscribe Headers into a Message

EmailEngine adds the headers itself. Set `listId` on the [submit request](/docs/api/post-v-1-account-account-submit) and send to one recipient at a time, and it adds `List-ID`, a signed `List-Unsubscribe` URL and `List-Unsubscribe-Post: List-Unsubscribe=One-Click` to the message:

```bash
curl -X POST "https://emailengine.example.com/v1/account/marketing/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {
      "name": "Newsletter Team",
      "address": "newsletter@example.com"
    },
    "to": [
      {
        "address": "subscriber@company.com"
      }
    ],
    "subject": "Weekly Newsletter",
    "html": "<p>Hello!</p><p>Here is your weekly newsletter.</p>",
    "listId": "weekly-newsletter"
  }'
```

Three conditions apply, and if any of them is not met the message is sent without the headers and can never produce this event:

- The `serviceUrl` setting must be set, because the unsubscribe URL is built from it
- `listId` must be in hostname format, such as `weekly-newsletter`
- The message must have exactly one `to` recipient. Use [mail merge](/docs/sending/mail-merge) to send a campaign, which submits one message per recipient

A recipient who is already on the suppression list for that `listId` is not sent to at all; the submit response reports the message as skipped. [Virtual Lists](/docs/advanced/virtual-lists) covers the whole flow, and [Blocklists](/docs/advanced/blocklists) covers managing the suppression list.

## Handling the Event

### Basic Handler

```javascript
async function handleListUnsubscribe(event) {
  const { account, date, data } = event;

  console.log(`Unsubscribe request for account ${account}`);
  console.log(`  Recipient: ${data.recipient}`);
  console.log(`  List ID: ${data.listId}`);
  console.log(`  Message-ID: ${data.messageId || 'none'}`);
  console.log(`  Unsubscribed at: ${date}`);

  // Update your mailing list database
  await removeFromMailingList({
    email: data.recipient,
    listId: data.listId,
    unsubscribedAt: new Date(date),
    source: 'emailengine'
  });
}
```

### Updating Subscription Database

```javascript
async function handleListUnsubscribe(event) {
  const { data, date } = event;

  // Update the subscriber's status in your database
  await db.subscribers.updateOne(
    { email: data.recipient.toLowerCase() },
    {
      $set: {
        [`lists.${data.listId}.subscribed`]: false,
        [`lists.${data.listId}.unsubscribedAt`]: new Date(date)
      },
      $push: {
        unsubscribeHistory: {
          listId: data.listId,
          messageId: data.messageId,
          timestamp: new Date(date),
          ipAddress: data.remoteAddress
        }
      }
    }
  );

  // Log for compliance purposes
  await db.unsubscribeLogs.insertOne({
    email: data.recipient,
    listId: data.listId,
    messageId: data.messageId,
    timestamp: new Date(date),
    ipAddress: data.remoteAddress,
    userAgent: data.userAgent
  });
}
```

### Syncing with External Mailing List Services

```javascript
async function handleListUnsubscribe(event) {
  const { data } = event;

  // Update your CRM or mailing list service
  switch (data.listId) {
    case 'weekly-newsletter':
      await mailchimpClient.updateMember(data.recipient, {
        status: 'unsubscribed'
      });
      break;

    case 'product-updates':
      await sendgridClient.removeFromList(
        'product-updates-list-id',
        data.recipient
      );
      break;

    default:
      // Generic unsubscribe handling
      await internalMailingService.unsubscribe(
        data.recipient,
        data.listId
      );
  }
}
```

### Tracking Unsubscribe Analytics

```javascript
async function handleListUnsubscribe(event) {
  const { data, date, account } = event;

  // Track unsubscribe metrics
  await analytics.track('email_unsubscribe', {
    account,
    listId: data.listId,
    email: data.recipient,
    messageId: data.messageId,
    timestamp: date
  });

  // Find which campaign triggered the unsubscribe
  if (data.messageId) {
    const originalMessage = await db.sentMessages.findOne({
      messageId: data.messageId
    });

    if (originalMessage) {
      // Increment unsubscribe count for the campaign
      await db.campaigns.updateOne(
        { _id: originalMessage.campaignId },
        { $inc: { unsubscribeCount: 1 } }
      );
    }
  }
}
```

## Technical Details

### The Unsubscribe Flow

1. **Sending**: a message submitted with `listId` to a single recipient gets `List-ID`, `List-Unsubscribe` and `List-Unsubscribe-Post` headers. The unsubscribe URL carries the account, list, recipient and Message-ID in a signed blob, so it cannot be forged or edited
2. **Recipient action**: the mail client sends a one-click `POST` to the URL, or the recipient opens it and confirms on the page
3. **Validation**: EmailEngine verifies the signature and that the account exists. A request that fails either check is answered with a plain acknowledgement rather than an error, as RFC 8058 asks, and nothing is recorded
4. **Suppression**: the recipient is added to the suppression list for that `listId`
5. **Webhook**: this event is sent if the address was not already on the list

The signed link carries no expiry. It stays valid for as long as the service secret it was signed with, so an unsubscribe from an old message still works.

### Suppression Lists

EmailEngine keeps one suppression list per `listId`. After an unsubscribe:

- Future submissions with the same `listId` to that recipient are skipped rather than sent, and the submit response says so
- Other lists are unaffected
- The entry stays until it is removed through the [Blocklists API](/docs/api/delete-v-1-blocklist-listid), the admin interface, or the recipient's own re-subscription, which sends [`listSubscribe`](/docs/webhooks/listsubscribe)

### Duplicate Detection

Only the request that adds the address to the suppression list fires the event. A second unsubscribe for the same address and list, from the same or another mail client, is acknowledged without a webhook.

## Best Practices

1. **Act on every unsubscribe** - Honor unsubscribe requests immediately to maintain compliance and sender reputation
2. **Update all systems** - Sync unsubscribe status across your CRM, mailing list service, and internal databases
3. **Log for compliance** - Maintain records of unsubscribe requests with timestamps and source information
4. **Use one list ID per list** - Distinct list IDs let a recipient leave one list without leaving the others
5. **Monitor unsubscribe rates** - Track unsubscribe rates per campaign to identify content or frequency issues
6. **Handle gracefully** - Even if your webhook processing fails, EmailEngine's suppression list already holds the entry and stops the sending

## Related Events

- [listSubscribe](/docs/webhooks/listsubscribe) - Triggered when a recipient re-subscribes from the unsubscribe page
- [messageSent](/docs/webhooks/messagesent) - Triggered when the original message was accepted for delivery
- [trackClick](/docs/webhooks/trackclick) - Triggered when links in messages are clicked

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Virtual Lists](/docs/advanced/virtual-lists) - Sending list mail with `listId` so that this event can fire
- [Blocklists](/docs/advanced/blocklists) - Reading and editing the suppression lists this event updates
- [Mail Merge](/docs/sending/mail-merge) - Sending one message per recipient

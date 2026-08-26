---
title: "trackOpen"
sidebar_position: 21
description: "Webhook event triggered when a recipient opens an email with open tracking enabled"
---

# trackOpen

The `trackOpen` webhook event reports that the tracking pixel of a sent message was loaded. EmailEngine adds that pixel to outgoing HTML messages when open tracking is on.

## When This Event is Triggered

The `trackOpen` event fires when a request reaches EmailEngine's `/open.gif` endpoint with a valid signature, which happens when the recipient's mail client loads the tracking pixel embedded in the message.

Open tracking works only when all of these hold:

1. The message had an HTML part. A plain text message gets no pixel
2. Open tracking was on for that message, either per submission or through the `trackOpens` setting (see [Enabling Open Tracking](#enabling-open-tracking))
3. A [`serviceUrl`](/docs/reference/configuration-options) was configured when the message was sent, since the pixel needs an absolute URL
4. The recipient's mail client loads remote images

## Limitations

Open tracking is an approximation, not a measurement:

- **Image blocking** - Many clients block remote images until the reader asks for them
- **Privacy proxies** - Apple Mail Privacy Protection and similar features fetch the pixel from a relay, so the open is recorded but the IP address and time belong to the proxy rather than the reader
- **Text-only viewing** - A reader who views the plain text alternative never loads the pixel
- **Caching** - A client that caches the image reports only the first open
- **Prefetching** - Security scanners and link-protection services fetch the pixel without a human involved

EmailEngine drops the requests it can recognize as automated, see [Automated request filtering](#automated-request-filtering). It cannot recognize all of them.

## Common Use Cases

- **Email engagement analytics** - Track open rates for marketing campaigns
- **Sales follow-up** - Know when a prospect has viewed your email
- **Support ticket monitoring** - Confirm when customers have seen your response
- **Delivery confirmation** - Verify that important messages were viewed
- **A/B testing** - Compare open rates across different subject lines or send times

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that sent the tracked message |
| `date` | string | Yes | ISO 8601 timestamp when the open was recorded |
| `event` | string | Yes | Always `trackOpen` |
| `data` | object | Yes | Open tracking data object (see below) |

The event carries no `path` or `specialUse`. The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | Message-ID header of the tracked email, as EmailEngine wrote it when the message was queued. It is the same value the Submit API returned and the [`messageSent`](/docs/webhooks/messagesent) event reports as `originalMessageId` |
| `remoteAddress` | string | Yes | IP address that requested the tracking pixel. Behind a reverse proxy this is the client address taken from `X-Forwarded-For`, and only when the request came from an address listed in `EENGINE_API_PROXY_ADDRESSES` |
| `userAgent` | string | No | `User-Agent` header of that request. Absent when the client sent none |

The event carries no folder, message or recipient reference beyond `messageId`. Record the Message-ID when you submit the message if you need to correlate an open with a recipient.

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:56:01.430Z",
  "event": "trackOpen",
  "data": {
    "messageId": "<0ee381d9-581a-2a57-6038-15e64c76f108@example.com>",
    "remoteAddress": "203.0.113.42",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36"
  }
}
```

### Without a User-Agent

Some clients fetch the image without a `User-Agent` header, and then the field is omitted:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "marketing",
  "date": "2025-10-17T14:30:00.000Z",
  "event": "trackOpen",
  "data": {
    "messageId": "<campaign-2025-q4-001@marketing.example.com>",
    "remoteAddress": "198.51.100.7"
  }
}
```

## Enabling Open Tracking

### Per-Message Tracking

Set `trackOpens` on the submission:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {
      "name": "Sales Team",
      "address": "sales@example.com"
    },
    "to": [
      {
        "address": "prospect@company.com"
      }
    ],
    "subject": "Your requested information",
    "html": "<p>Hello,</p><p>Here is the information you requested.</p>",
    "trackOpens": true
  }'
```

Messages submitted over the [SMTP interface](/docs/sending/smtp-interface) use the `X-EE-Tracking-Enabled` header instead, which switches both open and click tracking on or off for that message.

### Instance-Wide Setting

`trackOpens` is also a setting, applied to every submission that does not set the field itself:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "trackOpens": true,
    "trackClicks": false
  }'
```

In the admin interface the same switch is **Track Email Opens** under **Configuration > Email Processing**.

### Which Value Wins

EmailEngine resolves open tracking for each message in this order, taking the first value that is a boolean:

1. `trackOpens` on the submission
2. `trackingEnabled` on the submission, or the `X-EE-Tracking-Enabled` header for an SMTP submission. This covers both opens and clicks
3. The `trackOpens` setting
4. The `trackSentMessages` setting, which covers both opens and clicks and is kept for compatibility
5. Off

## Handling the Event

### Basic Handler

```javascript
async function handleTrackOpen(event) {
  const { account, date, data } = event;

  console.log(`Email opened for account ${account}`);
  console.log(`  Message-ID: ${data.messageId}`);
  console.log(`  Opened at: ${date}`);
  console.log(`  From IP: ${data.remoteAddress}`);

  // Update your database or analytics system
  await recordEmailOpen({
    messageId: data.messageId,
    openedAt: new Date(date),
    ipAddress: data.remoteAddress,
    userAgent: data.userAgent
  });
}
```

### Tracking Multiple Opens

Since the same email may be opened multiple times by the same recipient (or the tracking pixel may be cached), consider deduplication:

```javascript
const recentOpens = new Map();

async function handleTrackOpen(event) {
  const { data, date } = event;
  const cacheKey = `${data.messageId}:${data.remoteAddress}`;

  // Check if we've seen this open recently (within 1 hour)
  const lastOpen = recentOpens.get(cacheKey);
  if (lastOpen && (Date.now() - lastOpen) < 3600000) {
    console.log('Duplicate open detected, skipping');
    return;
  }

  recentOpens.set(cacheKey, Date.now());

  // Process the open event
  await recordEmailOpen(event);
}
```

### Correlating Opens with Sent Messages

Use the `messageId` to link opens back to your original sent messages:

```javascript
async function handleTrackOpen(event) {
  const { data } = event;

  // Find the original message in your database
  const sentMessage = await db.sentMessages.findOne({
    messageId: data.messageId
  });

  if (sentMessage) {
    // Update open status
    await db.sentMessages.updateOne(
      { messageId: data.messageId },
      {
        $set: {
          opened: true,
          openedAt: new Date(event.date)
        },
        $inc: { openCount: 1 }
      }
    );

    // Notify sales team if this is a prospect email
    if (sentMessage.type === 'prospect') {
      await notifySalesTeam(sentMessage, event);
    }
  }
}
```

## Technical Details

### How the Tracking Pixel Works

When open tracking applies to a message, EmailEngine, in each HTML part of the outgoing message:

1. Builds a payload naming the account and the Message-ID, base64url encodes it as the `data` parameter, and signs it with the instance's `serviceSecret` as the `sig` parameter
2. Inserts a 1x1 pixel image tag just before the closing `</body>` tag, or appends it when the part has no `</body>`

The pixel it inserts:

```html
<img src="https://emailengine.example.com/open.gif?data=eyJhY3QiOiJvcGVuIn0&sig=Zm9vYmFy" style="border: 0px; width:1px; height: 1px;" tabindex="-1" width="1" height="1" alt="">
```

`/open.gif` verifies the signature before doing anything else and answers `403` when it does not match, so a tracking URL cannot be forged or edited to name another account. EmailEngine mints a `serviceSecret` on its own the first time it needs one, so signing is always on; set your own value through the [settings API](/docs/api/post-v-1-settings) if you want to control it.

A request that passes verification is answered with a 1x1 transparent GIF and `Cache-Control: no-store`, whether or not a webhook was sent.

### Automated Request Filtering

Before sending the webhook, EmailEngine checks the requesting address against:

- The published address ranges of Google's crawlers, which is what fetches the pixel when a Gmail user has not opened the message
- A reverse DNS lookup, treating a hostname under `barracuda.com` or `spfbl.net` as a scanner

A request that matches is written to the log at debug level and no webhook is sent. Everything else is reported, including privacy relays and link-protection scanners that do not identify themselves.

## Best Practices

1. **Do not rely on opens alone** - Image blocking and privacy relays make the count a floor, not a measurement
2. **Combine with click tracking** - Use both open and click tracking for better engagement insights
3. **Handle duplicates** - The same email may trigger multiple open events
4. **Respect privacy** - Be transparent with recipients about tracking and comply with privacy regulations
5. **Use for trends, not absolutes** - Open rates are best used for comparing relative performance, not absolute engagement
6. **Consider time zones** - Analyze open times to optimize send times for your audience

## Related Events

- [trackClick](/docs/webhooks/trackclick) - Triggered when a tracked link is clicked
- [listUnsubscribe](/docs/webhooks/listunsubscribe) - Triggered when a recipient uses the one-click unsubscribe link
- [messageSent](/docs/webhooks/messagesent) - Carries the `messageId` this event refers back to

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [trackClick](/docs/webhooks/trackclick) - The click half of the same tracking configuration
- [Sending Emails](/docs/sending/basic-sending) - Submitting a message with `trackOpens` set
- [SMTP interface](/docs/sending/smtp-interface) - Switching tracking on with `X-EE-Tracking-Enabled`
- [Settings API](/docs/api/post-v-1-settings) - Setting `trackOpens` and `serviceSecret`

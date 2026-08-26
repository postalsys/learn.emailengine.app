---
title: "trackClick"
sidebar_position: 22
description: "Webhook event triggered when a recipient clicks a tracked link in an email"
---

# trackClick

The `trackClick` webhook event reports that a rewritten link in a sent message was followed. EmailEngine rewrites the links of outgoing HTML messages through its own `/redirect` endpoint when click tracking is on.

## When This Event is Triggered

The `trackClick` event fires when a request reaches EmailEngine's `/redirect` endpoint with a valid signature, which happens when the recipient follows a rewritten link. EmailEngine records the click and then redirects the browser to the original destination.

Click tracking works only when all of these hold:

1. The message had an HTML part with `http:` or `https:` links. Links in the plain text alternative are never rewritten
2. Click tracking was on for that message, either per submission or through the `trackClicks` setting (see [Enabling Click Tracking](#enabling-click-tracking))
3. A [`serviceUrl`](/docs/reference/configuration-options) was configured when the message was sent, since the rewritten link needs an absolute URL

## Limitations

- **Plain text parts** - Only HTML parts are rewritten, so a reader in plain text mode follows the original link and nothing is recorded
- **Non-HTTP schemes** - `mailto:`, `tel:` and other schemes are left alone
- **Link prefetching** - Security scanners and link-protection services follow links without a human involved
- **Visible rewriting** - The recipient sees an EmailEngine URL in the status bar rather than the destination, and some corporate mail filters rewrite it again on top

EmailEngine drops the requests it can recognize as automated, see [Automated request filtering](#automated-request-filtering). It cannot recognize all of them.

## Common Use Cases

- **Email engagement analytics** - Track click-through rates for marketing campaigns
- **Link performance analysis** - Identify which links in your emails get the most engagement
- **Sales follow-up** - Know when a prospect clicks on a proposal link
- **Content optimization** - Determine which call-to-action buttons perform best
- **A/B testing** - Compare click rates across different email variations
- **User journey tracking** - Understand recipient behavior by tracking link interactions

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that sent the tracked message |
| `date` | string | Yes | ISO 8601 timestamp when the click was recorded |
| `event` | string | Yes | Always `trackClick` |
| `data` | object | Yes | Click tracking data object (see below) |

The event carries no `path` or `specialUse`. The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | Message-ID header of the tracked email, as EmailEngine wrote it when the message was queued. It is the same value the Submit API returned and the [`messageSent`](/docs/webhooks/messagesent) event reports as `originalMessageId` |
| `url` | string | Yes | The destination URL, exactly as it appeared in the message before rewriting |
| `remoteAddress` | string | Yes | IP address that followed the link. Behind a reverse proxy this is the client address taken from `X-Forwarded-For`, and only when the request came from an address listed in `EENGINE_API_PROXY_ADDRESSES` |
| `userAgent` | string | No | `User-Agent` header of that request. Absent when the client sent none |

The event carries no recipient reference. Record the Message-ID when you submit the message if you need to correlate a click with a recipient.

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:56:27.882Z",
  "event": "trackClick",
  "data": {
    "messageId": "<0ee381d9-581a-2a57-6038-15e64c76f108@example.com>",
    "url": "https://example.com/product-page",
    "remoteAddress": "203.0.113.42",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36"
  }
}
```

### Without a User-Agent

Some clients follow the link without a `User-Agent` header, and then the field is omitted:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "marketing",
  "date": "2025-10-17T14:32:15.000Z",
  "event": "trackClick",
  "data": {
    "messageId": "<campaign-2025-q4-001@marketing.example.com>",
    "url": "https://shop.example.com/promo?code=SAVE20",
    "remoteAddress": "198.51.100.7"
  }
}
```

## Enabling Click Tracking

### Per-Message Tracking

Set `trackClicks` on the submission:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {
      "name": "Marketing Team",
      "address": "marketing@example.com"
    },
    "to": [
      {
        "address": "customer@company.com"
      }
    ],
    "subject": "Check out our latest products",
    "html": "<p>Hello!</p><p>Visit our <a href=\"https://shop.example.com\">online store</a>.</p>",
    "trackClicks": true
  }'
```

Messages submitted over the [SMTP interface](/docs/sending/smtp-interface) use the `X-EE-Tracking-Enabled` header instead, which switches both open and click tracking on or off for that message.

### Instance-Wide Setting

`trackClicks` is also a setting, applied to every submission that does not set the field itself:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "trackClicks": true,
    "trackOpens": false
  }'
```

In the admin interface the same switch is **Track Link Clicks** under **Configuration > Email Processing**.

### Which Value Wins

EmailEngine resolves click tracking for each message in this order, taking the first value that is a boolean:

1. `trackClicks` on the submission
2. `trackingEnabled` on the submission, or the `X-EE-Tracking-Enabled` header for an SMTP submission. This covers both opens and clicks
3. The `trackClicks` setting
4. The `trackSentMessages` setting, which covers both opens and clicks and is kept for compatibility
5. Off

## Handling the Event

### Basic Handler

```javascript
async function handleTrackClick(event) {
  const { account, date, data } = event;

  console.log(`Link clicked for account ${account}`);
  console.log(`  Message-ID: ${data.messageId}`);
  console.log(`  URL: ${data.url}`);
  console.log(`  Clicked at: ${date}`);
  console.log(`  From IP: ${data.remoteAddress}`);

  // Update your database or analytics system
  await recordLinkClick({
    messageId: data.messageId,
    clickedUrl: data.url,
    clickedAt: new Date(date),
    ipAddress: data.remoteAddress,
    userAgent: data.userAgent
  });
}
```

### Tracking Click Patterns

Analyze which links perform best across your campaigns:

```javascript
async function handleTrackClick(event) {
  const { data, date } = event;

  // Parse the URL to categorize the click
  const clickedUrl = new URL(data.url);

  // Track clicks by domain and path
  await analytics.trackEvent('email_link_click', {
    messageId: data.messageId,
    url: data.url,
    domain: clickedUrl.hostname,
    path: clickedUrl.pathname,
    timestamp: date
  });

  // Track specific call-to-action buttons
  if (clickedUrl.pathname.includes('/buy') || clickedUrl.pathname.includes('/purchase')) {
    await analytics.trackConversion('purchase_click', {
      messageId: data.messageId,
      timestamp: date
    });
  }
}
```

### Correlating Clicks with Sent Messages

Use the `messageId` to link clicks back to your original sent messages:

```javascript
async function handleTrackClick(event) {
  const { data } = event;

  // Find the original message in your database
  const sentMessage = await db.sentMessages.findOne({
    messageId: data.messageId
  });

  if (sentMessage) {
    // Update click stats
    await db.sentMessages.updateOne(
      { messageId: data.messageId },
      {
        $set: { hasClicks: true },
        $inc: { clickCount: 1 },
        $push: {
          clicks: {
            url: data.url,
            clickedAt: new Date(event.date),
            ipAddress: data.remoteAddress
          }
        }
      }
    );

    // Notify sales team if this is a high-value link
    if (data.url.includes('/pricing') || data.url.includes('/demo')) {
      await notifySalesTeam(sentMessage, event);
    }
  }
}
```

## Technical Details

### How Link Rewriting Works

When click tracking applies to a message, EmailEngine, in each HTML part of the outgoing message:

1. Finds every `<a href="http...">` attribute
2. Builds a payload naming the account, the Message-ID and the original URL, base64url encodes it as the `data` parameter, and signs it with the instance's `serviceSecret` as the `sig` parameter
3. Replaces the `href` with a `/redirect` URL carrying those two parameters

A rewritten link:

```html
<a href="https://emailengine.example.com/redirect?data=eyJhY3QiOiJjbGljayJ9&amp;sig=Zm9vYmFy">View Product</a>
```

`/redirect` verifies the signature before doing anything else and answers `403` when it does not match, so a tracking URL cannot be edited to redirect somewhere else. EmailEngine mints a `serviceSecret` on its own the first time it needs one, so signing is always on; set your own value through the [settings API](/docs/api/post-v-1-settings) if you want to control it.

### Links That Are Not Rewritten

Two kinds of link are left as they are, so that tracking cannot break them:

- A link to this instance's own `/unsubscribe` endpoint, which carries its own signed payload
- A link to this instance's own `/redirect` endpoint, so a forwarded or re-sent message is not wrapped twice

Both are recognized by origin, so a `/redirect` URL on a different host is rewritten like any other link.

### Redirect Flow

When a recipient follows a rewritten link:

1. The browser requests `/redirect` with the `data` and `sig` parameters
2. EmailEngine verifies the signature and rejects the request with `403` if it does not match
3. EmailEngine checks whether the request looks automated, see below
4. If it does not, the `trackClick` webhook is queued
5. The browser is answered with a `302` redirect to the original URL, whether or not a webhook was queued

### Automated Request Filtering

Before queuing the webhook, EmailEngine checks the requesting address against:

- The published address ranges of Google's crawlers
- A reverse DNS lookup, treating a hostname under `barracuda.com` or `spfbl.net` as a scanner

A request that matches is written to the log at debug level and no webhook is sent, but the redirect still happens. Everything else is reported, including link-protection scanners that do not identify themselves.

## Best Practices

1. **Use with open tracking** - Opens and clicks answer different questions; enable both to tell a read from an act
2. **Handle multiple clicks** - The same link may be clicked multiple times; decide how to count them
3. **Respect privacy** - Be transparent with recipients about tracking and comply with privacy regulations (GDPR, CAN-SPAM)
4. **Set your own service secret** - EmailEngine mints one if you do not, but a value you set is one you can rotate and keep across restores
5. **Monitor for anomalies** - Watch for unusual click patterns that might indicate security scanner activity
6. **Test tracked links** - Verify that tracked links redirect correctly to the intended destinations

## Related Events

- [trackOpen](/docs/webhooks/trackopen) - Triggered when the tracking pixel of a message is loaded
- [listUnsubscribe](/docs/webhooks/listunsubscribe) - Triggered when a recipient uses the one-click unsubscribe link
- [messageSent](/docs/webhooks/messagesent) - Carries the `messageId` this event refers back to

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [trackOpen](/docs/webhooks/trackopen) - The open half of the same tracking configuration
- [Sending Emails](/docs/sending/basic-sending) - Submitting a message with `trackClicks` set
- [SMTP interface](/docs/sending/smtp-interface) - Switching tracking on with `X-EE-Tracking-Enabled`
- [Settings API](/docs/api/post-v-1-settings) - Setting `trackClicks` and `serviceSecret`

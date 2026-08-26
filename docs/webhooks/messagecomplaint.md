---
title: "messageComplaint"
sidebar_position: 11
description: "Webhook event triggered when an FBL (Feedback Loop) complaint is detected"
---

# messageComplaint

The `messageComplaint` webhook event is triggered when EmailEngine detects a feedback loop (FBL) complaint in a monitored mailbox. This event helps you identify when recipients have marked your emails as spam or unwanted, allowing you to maintain sender reputation and comply with email best practices.

## When This Event is Triggered

The `messageComplaint` event fires when a message arriving in the Inbox is recognized as an abuse report and the report names at least one complaining recipient (`Original-Rcpt-To`, or the Hotmail `X-HmXmrOriginalRecipient` header). A report without a recipient produces no event.

A message is checked for complaint content when one of these holds:

- It has a `message/feedback-report` part (an ARF report, [RFC 5965](https://www.rfc-editor.org/rfc/rfc5965))
- It comes from `staff@hotmail.com`, embeds the original message as `message/rfc822` or `message/rfc822-headers`, and has `complaint` in the subject (Microsoft JMRP reports)

Messages in other folders are not checked.

The complaint message itself also produces a [`messageNew`](/docs/webhooks/messagenew) event, sent before this one, with `isComplaint: true` and `relatedMessageId` set to the Message-ID of the reported message when the report included it. On IMAP accounts neither event is sent for messages dated before the account's `notifyFrom`.

## Common Use Cases

- **List hygiene** - Automatically unsubscribe users who report spam
- **Reputation management** - Track complaint rates to maintain good sender reputation
- **Deliverability monitoring** - Identify content or sending patterns causing complaints
- **Compliance** - Fulfill legal requirements to honor unsubscribe requests
- **Analytics** - Build dashboards showing complaint trends by campaign or domain
- **Blocklist prevention** - Address issues before reaching ISP complaint thresholds

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that received the complaint message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `messageComplaint` |
| `data` | object | Yes | Complaint data object (see below) |

The event carries no `path` or `specialUse`. The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Complaint Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `complaintMessage` | string | Yes | EmailEngine message ID of the complaint notification email itself. Fetch it through the [message API](/docs/api/get-v-1-account-account-message-message) to read the full report |
| `arf` | object | Yes | Fields of the feedback report, camelCased (see below) |
| `headers` | object | No | Selected headers of the reported message, when the report embedded it (see below) |

### ARF Object Structure

Every field of the `message/feedback-report` part is passed through under its camelCased name, so the exact set depends on the reporter. These fields are single strings:

| Field | Description |
|-------|-------------|
| `feedbackType` | Type of report as stated by the reporter, typically `abuse` |
| `userAgent` | Software that generated the report |
| `version` | ARF format version, typically `1` |
| `originalEnvelopeId` | Envelope ID of the original delivery |
| `originalMailFrom` | Envelope sender of the original message, without angle brackets |
| `abuseType` | Specific abuse type, for example `complaint` |
| `arrivalDate` | When the original message arrived, as an ISO 8601 timestamp. `Received-Date` is reported under this name as well |
| `reportingMta` | MTA that generated the report, as written in the report (for example `dns; mx.isp.example.com`) |
| `sourceIp` | IP address the original message was sent from |
| `source` | Name of the reporting system |
| `subscriptionLink` | Link to subscription preferences |
| `incidents` | Number of incidents the report covers |

Any other field, including `originalRcptTo`, is an array of strings with one entry per occurrence. `originalRcptTo` lists the recipients who complained and is the field that decides whether the event is sent at all. Empty fields are omitted, and an `Arrival-Date` that does not parse is dropped.

For Microsoft JMRP reports EmailEngine fills in what the report leaves out: `source` is set to `Hotmail`, `feedbackType` to `abuse` and `abuseType` to `complaint` unless the report says otherwise, and `originalMailFrom` falls back to the `Return-Path` of the embedded message.

### Headers Object Structure

When the report embeds the reported message (`message/rfc822`, `message/rfc822-headers`, `text/rfc822-headers` or `text/rfc822-header`), `headers` carries these values from it. Fields the embedded message does not have are omitted, and the object is absent when nothing was found:

| Field | Type | Description |
|-------|------|-------------|
| `messageId` | string | Message-ID of the reported message |
| `from` | string | Sender address |
| `to` | array | Recipient addresses |
| `cc` | array | CC addresses |
| `bcc` | array | BCC addresses |
| `subject` | string | Subject line, MIME decoded |
| `date` | string | Date header as an ISO 8601 timestamp |

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T07:06:11.697Z",
  "event": "messageComplaint",
  "data": {
    "complaintMessage": "AAAAAQAABzE",
    "arf": {
      "source": "Hotmail",
      "feedbackType": "abuse",
      "abuseType": "complaint",
      "originalMailFrom": "sender@example.com",
      "originalRcptTo": ["recipient@hotmail.com"],
      "sourceIp": "203.0.113.42",
      "arrivalDate": "2025-10-17T07:06:35.021Z"
    },
    "headers": {
      "messageId": "<57f34982-43cc-6534-40f9-0f72f1c8a158@example.com>",
      "from": "sender@example.com",
      "to": ["recipient@hotmail.com"],
      "subject": "Your weekly newsletter",
      "date": "2025-10-17T07:06:34.000Z"
    }
  }
}
```

## ARF Parsing Details

EmailEngine parses Abuse Reporting Format (ARF) messages according to RFC 5965 and the Microsoft JMRP variant.

### Parsed Content Types

EmailEngine examines message attachments for these content types:

| Content-Type | Purpose |
|--------------|---------|
| `message/feedback-report` | Standard ARF feedback report with structured fields |
| `message/rfc822` | Complete original message; only its headers are read |
| `message/rfc822-headers` | Original message headers only |
| `text/rfc822-headers` | Original message headers (alternate type) |
| `text/rfc822-header` | Original message headers (singular form) |

Only the first three are downloaded for parsing. A report that puts the original headers in a `text/rfc822-headers` or `text/rfc822-header` part is recognized but contributes nothing, so its `headers` object is absent and the event is sent only if the feedback report itself named a recipient.

### Extracted Original Message Headers

From the embedded original message, EmailEngine reads:

| Header | Webhook Field | Description |
|--------|---------------|-------------|
| `Message-ID` | `headers.messageId` | Message-ID of the reported message |
| `From` | `headers.from` | Sender address |
| `To` | `headers.to` | Recipient addresses (array) |
| `CC` | `headers.cc` | CC addresses (array) |
| `BCC` | `headers.bcc` | BCC addresses (array) |
| `Subject` | `headers.subject` | Subject line (MIME decoded) |
| `Date` | `headers.date` | Message date (ISO 8601) |
| `Return-Path` | `arf.originalMailFrom` | Envelope sender, JMRP reports only, when the report has no `Original-Mail-From` |
| `X-HmXmrOriginalRecipient` | `arf.originalRcptTo` | Original recipient (Hotmail), appended to the list |
| `X-Sender-IP` | `arf.sourceIp` | Sender IP (Microsoft), when the report has no `Source-IP` |
| `X-MS-Exchange-CrossTenant-OriginalArrivalTime` | `arf.arrivalDate` | Arrival time (Microsoft), when the report has no `Arrival-Date` |

### Example: Standard ARF Message

A standard RFC 5965 complaint message looks like this:

```text
From: abusedesk@isp.example.com
To: abuse@sender.example.com
Subject: FBL Notification
Content-Type: multipart/report; report-type=feedback-report; boundary=boundary

--boundary
Content-Type: text/plain

This is a spam complaint from one of our users.

--boundary
Content-Type: message/feedback-report

Feedback-Type: abuse
User-Agent: ISP-FBL/1.0
Version: 1
Original-Mail-From: newsletter@sender.example.com
Original-Rcpt-To: user@isp.example.com
Arrival-Date: Thu, 17 Oct 2025 07:06:35 +0000
Source-IP: 203.0.113.42
Reporting-MTA: dns; mx.isp.example.com

--boundary
Content-Type: message/rfc822

From: newsletter@sender.example.com
To: user@isp.example.com
Subject: Your Weekly Newsletter
Date: Thu, 17 Oct 2025 07:06:34 +0000
Message-ID: <abc123@sender.example.com>

The original message body.

--boundary--
```

EmailEngine parses this into:

```json
{
  "arf": {
    "feedbackType": "abuse",
    "userAgent": "ISP-FBL/1.0",
    "version": "1",
    "originalMailFrom": "newsletter@sender.example.com",
    "originalRcptTo": ["user@isp.example.com"],
    "arrivalDate": "2025-10-17T07:06:35.000Z",
    "sourceIp": "203.0.113.42",
    "reportingMta": "dns; mx.isp.example.com"
  },
  "headers": {
    "messageId": "<abc123@sender.example.com>",
    "from": "newsletter@sender.example.com",
    "to": ["user@isp.example.com"],
    "subject": "Your Weekly Newsletter",
    "date": "2025-10-17T07:06:34.000Z"
  }
}
```

### Example: Hotmail/JMRP Complaint

Microsoft's JMRP complaints have no feedback report part. The original recipient and the sending IP are in headers of the embedded message:

```text
From: staff@hotmail.com
To: abuse@sender.example.com
Subject: complaint about message from 203.0.113.42
Content-Type: multipart/report; report-type=feedback-report; boundary=boundary

--boundary
Content-Type: text/plain

This is a complaint from a Hotmail user.

--boundary
Content-Type: message/rfc822-headers

Return-Path: <newsletter@sender.example.com>
From: newsletter@sender.example.com
To: user@hotmail.com
Subject: Your Weekly Newsletter
Date: Thu, 17 Oct 2025 07:06:34 +0000
Message-ID: <abc123@sender.example.com>
X-HmXmrOriginalRecipient: user@hotmail.com
X-Sender-IP: 203.0.113.42
X-MS-Exchange-CrossTenant-OriginalArrivalTime: 17 Oct 2025 07:06:35.0210 (UTC)

--boundary--
```

EmailEngine recognizes the `staff@hotmail.com` sender and reports:

```json
{
  "arf": {
    "source": "Hotmail",
    "feedbackType": "abuse",
    "abuseType": "complaint",
    "originalMailFrom": "newsletter@sender.example.com",
    "originalRcptTo": ["user@hotmail.com"],
    "sourceIp": "203.0.113.42",
    "arrivalDate": "2025-10-17T07:06:35.021Z"
  },
  "headers": {
    "messageId": "<abc123@sender.example.com>",
    "from": "newsletter@sender.example.com",
    "to": ["user@hotmail.com"],
    "subject": "Your Weekly Newsletter",
    "date": "2025-10-17T07:06:34.000Z"
  }
}
```

### Handling Incomplete ARF Data

Not all complaint messages contain complete ARF data. EmailEngine extracts whatever is available, so check for the presence of fields before using them:

```javascript
async function handleComplaint(event) {
  const { arf, headers } = event.data;

  const complainants = arf?.originalRcptTo || [];
  const originalMessageId = headers?.messageId;
  const source = arf?.source || 'unknown';

  if (complainants.length === 0) {
    console.warn('Complaint received but no recipient identified');
    return;
  }

  await processComplaint(event.data);
}
```

## Understanding FBL Complaints

Recipients mark emails as spam for various reasons:

- **Unwanted emails** - User no longer wants to receive messages
- **Forgotten subscription** - User doesn't remember signing up
- **Difficult unsubscribe** - Easier to click "spam" than find unsubscribe link
- **Misleading content** - Email doesn't match user expectations
- **Excessive frequency** - Too many emails sent too often

Mailbox providers deliver these reports through feedback loop programs that a sender enrolls in, such as Microsoft's Junk Mail Reporting Program (JMRP) for Outlook.com and Hotmail. The reports arrive at the address registered with the program, so that mailbox has to be one EmailEngine monitors.

## Handling the Event

### Basic Handler

```javascript
async function handleMessageComplaint(event) {
  const { account, data } = event;

  console.log(`Complaint detected for account ${account}:`);
  console.log(`  Complaint Message ID: ${data.complaintMessage}`);

  if (data.arf) {
    console.log(`  Source: ${data.arf.source}`);
    console.log(`  Feedback Type: ${data.arf.feedbackType}`);
    console.log(`  Complainants: ${data.arf.originalRcptTo?.join(', ')}`);
  }

  if (data.headers) {
    console.log(`  Original Message-ID: ${data.headers.messageId}`);
    console.log(`  Original Subject: ${data.headers.subject}`);
  }

  await processComplaint(data);
}
```

### Automatic Unsubscribe

```javascript
async function processComplaint(complaintData) {
  const { arf, headers } = complaintData;

  const complainants = arf?.originalRcptTo || [];

  for (const email of complainants) {
    await db.subscriptions.updateMany(
      { email: email.toLowerCase() },
      {
        $set: {
          subscribed: false,
          unsubscribeReason: 'spam_complaint',
          unsubscribedAt: new Date(),
          complaintSource: arf?.source
        }
      }
    );

    await db.suppressionList.upsert({
      email: email.toLowerCase(),
      reason: 'complaint',
      source: arf?.source,
      originalMessageId: headers?.messageId,
      createdAt: new Date()
    });

    console.log(`Unsubscribed ${email} due to spam complaint`);
  }
}
```

### Tracking Complaint Metrics

```javascript
async function trackComplaintMetrics(event) {
  const { account, data } = event;

  const campaignId = extractCampaignId(data.headers?.messageId);

  await metrics.increment('email.complaints', {
    account,
    source: data.arf?.source || 'unknown',
    feedbackType: data.arf?.feedbackType || 'unknown',
    campaign: campaignId
  });

  const stats = await getRecentStats(account);
  const complaintRate = stats.complaints / stats.totalSent;

  if (complaintRate > 0.001) {
    await sendAlert({
      type: 'high_complaint_rate',
      account,
      rate: complaintRate,
      threshold: 0.001
    });
  }
}

function extractCampaignId(messageId) {
  const match = messageId?.match(/campaign-([a-z0-9]+)/i);
  return match ? match[1] : null;
}
```

### Correlating with Original Message

```javascript
async function correlateComplaint(complaintData) {
  const { headers, arf } = complaintData;

  let originalMessage = null;

  if (headers?.messageId) {
    originalMessage = await db.sentMessages.findOne({
      messageId: headers.messageId
    });
  }

  if (!originalMessage && arf?.originalMailFrom && arf?.arrivalDate) {
    originalMessage = await db.sentMessages.findOne({
      from: arf.originalMailFrom,
      sentAt: {
        $gte: new Date(Date.parse(arf.arrivalDate) - 86400000),
        $lte: new Date(arf.arrivalDate)
      }
    });
  }

  if (originalMessage) {
    await db.sentMessages.updateOne(
      { _id: originalMessage._id },
      {
        $set: { complained: true },
        $push: {
          complaints: {
            date: new Date(),
            recipients: arf?.originalRcptTo,
            source: arf?.source
          }
        }
      }
    );
  }

  return originalMessage;
}
```

## Best Practices

1. **Immediately unsubscribe complainants** - Honor complaints instantly to maintain sender reputation
2. **Add to suppression list** - Prevent sending to complainants across all campaigns
3. **Monitor complaint rates** - Track rates per campaign and overall; investigate spikes
4. **Review complained content** - Analyze what content generates complaints
5. **Improve list acquisition** - Ensure clear opt-in and set expectations
6. **Make unsubscribe easy** - Prominent, one-click unsubscribe reduces complaints
7. **Respect frequency preferences** - Allow users to control email frequency
8. **Clean inactive subscribers** - Remove users who have not engaged for a long time

## Related Events

- [messageBounce](/docs/webhooks/messagebounce) - Triggered when a bounce notification is received
- [messageFailed](/docs/webhooks/messagefailed) - Triggered when EmailEngine gives up on a queued email
- [messageSent](/docs/webhooks/messagesent) - Triggered when a message is accepted for delivery
- [messageNew](/docs/webhooks/messagenew) - The complaint notification also triggers this event

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [listUnsubscribe](/docs/webhooks/listunsubscribe) - The event EmailEngine's own one-click unsubscribe produces
- [Sending Emails](/docs/sending/basic-sending) - How to send emails through EmailEngine
- [Message API](/docs/api/get-v-1-account-account-message-message) - Fetching the report message by `complaintMessage`
- [Settings API](/docs/api/post-v-1-settings) - Configure webhook settings

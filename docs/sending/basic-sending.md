---
title: Basic Email Sending
sidebar_position: 2
description: Learn how to send emails using EmailEngine's submit API with HTML, attachments, and custom headers
---

<!--
SOURCE ATTRIBUTION:
- Primary: blog/2025-01-08-sending-an-email-from-emailengine.md
- Merged: docs/usage/sending-emails.md
-->

# Basic Email Sending

EmailEngine simplifies sending emails through registered accounts' SMTP servers. This guide covers the fundamentals of sending emails using the [submit API](/docs/api/post-v-1-account-account-submit).

## Why It Matters

When your application sends on behalf of a customer, talking SMTP directly means handling each provider's authentication, rate limits, retry rules, and error codes. EmailEngine puts one REST endpoint in front of the customer's mailbox instead, with the same JSON response shape whichever provider is behind it, and a queue that retries failed submissions.

## Step-by-Step Guide

### 1. Register the Account

Before sending, register an email account in EmailEngine using the [account registration API](/docs/api/post-v-1-account).

**Endpoint:** `POST /v1/account`

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account" \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "example",
    "name": "Andris Reinman",
    "email": "andris@example.com",
    "imap": {
      "auth": { "user": "andris", "pass": "secretpass" },
      "host": "mail.example.com",
      "port": 993,
      "secure": true
    },
    "smtp": {
      "auth": { "user": "andris", "pass": "secretpass" },
      "host": "mail.example.com",
      "port": 465,
      "secure": true
    }
  }'
```

**Expected response:**

```json
{
  "account": "example",
  "state": "new"
}
```

**Important**: If you use an SMTP port other than 465, set `"secure": false`.

### 2. Wait for Connection

Before sending, ensure the account is connected. Poll the account status:

```bash
curl "http://127.0.0.1:3000/v1/account/example" \
  -H "Authorization: Bearer <your-token>"
```

Wait until `state` becomes `"connected"`. Submission is rejected while EmailEngine is performing the initial sync.

### 3. Submit a Simple Email

**Endpoint:** `POST /v1/account/:id/submit`

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/example/submit" \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [
      {
        "name": "Recipient Name",
        "address": "recipient@example.com"
      }
    ],
    "subject": "Test message",
    "text": "Hello from myself!",
    "html": "<p>Hello from myself!</p>"
  }'
```

**Expected response (queued, not yet delivered):**

```json
{
  "response": "Queued for delivery",
  "messageId": "<99f7f0ec-90a1-caaf-698b-18e096c7679e@example.com>",
  "sendAt": "2025-05-14T10:22:31.312Z",
  "queueId": "4646ac53857fd2b2"
}
```

The message is now queued for delivery. EmailEngine will handle the actual SMTP transmission.

## Message Components

### Recipients

You can specify multiple recipients in `to`, `cc`, and `bcc` fields:

```json
{
  "to": [
    { "name": "Alice", "address": "alice@example.com" },
    { "name": "Bob", "address": "bob@example.com" }
  ],
  "cc": [{ "address": "manager@example.com" }],
  "bcc": [{ "address": "archive@example.com" }]
}
```

The `name` field is optional but recommended for a better recipient experience.

### Content

#### Plain Text and HTML

Always provide both plain text and HTML versions for best compatibility:

```json
{
  "subject": "Welcome to our service",
  "text": "Welcome! Visit https://example.com to get started.",
  "html": "<p>Welcome!</p><p>Visit <a href='https://example.com'>our site</a> to get started.</p>"
}
```

#### HTML Only

If you only provide HTML, EmailEngine can auto-generate a plain text version:

```json
{
  "subject": "HTML Newsletter",
  "html": "<h1>Hello!</h1><p>This is an HTML email.</p>"
}
```

### Attachments

Add attachments using the `attachments` array:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Document attached",
  "text": "Please find the document attached.",
  "attachments": [
    {
      "filename": "document.pdf",
      "content": "base64-encoded-content-here",
      "contentType": "application/pdf"
    }
  ]
}
```

**Attachment options:**

- `filename` - Attachment filename (optional, recommended)
- `content` - Base64 encoded content (required unless using `reference`)
- `contentType` - MIME type (optional, auto-detected if omitted)
- `cid` - Content ID for inline images (optional)
- `reference` - Reference an existing attachment by ID instead of providing content (optional)

#### Inline Images

Reference inline images in HTML using Content ID:

```json
{
  "html": "<p>Logo: <img src='cid:logo' /></p>",
  "attachments": [
    {
      "filename": "logo.png",
      "content": "iVBORw0KGgoAAAANS...",
      "contentType": "image/png",
      "cid": "logo"
    }
  ]
}
```

### Custom Headers

Add custom email headers:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Test",
  "text": "Test message",
  "headers": {
    "X-Custom-Header": "value",
    "X-Campaign-ID": "campaign-123",
    "List-Unsubscribe": "<mailto:unsubscribe@example.com>"
  }
}
```

Common custom headers:

- `X-Custom-*` - Your custom headers
- `Reply-To` - Set reply address
- `List-Unsubscribe` - Unsubscribe link for bulk email
- `X-Priority` - Set message priority (1-5)

### Sender Information

Override default sender information:

```json
{
  "from": {
    "name": "Support Team",
    "address": "support@example.com"
  },
  "replyTo": {
    "name": "No Reply",
    "address": "noreply@example.com"
  },
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Support Ticket Response",
  "text": "Your ticket has been updated."
}
```

If omitted, EmailEngine uses the account's configured email and name.

## Advanced Options

### Scheduled Sending

Schedule an email for future delivery:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Scheduled message",
  "text": "This will be sent at the specified time",
  "sendAt": "2025-12-25T09:00:00.000Z"
}
```

The message stays in the queue until the specified time.

### Skip Sent Folder

Prevent saving a copy to the Sent Mail folder:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "No copy saved",
  "text": "This won't appear in Sent Mail",
  "copy": false
}
```

Useful for bulk sending to avoid cluttering the Sent folder.

### Custom Message ID

Specify your own Message-ID for threading:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Custom ID",
  "text": "Message with custom ID",
  "messageId": "<custom-id-12345@example.com>"
}
```

Important for maintaining email threads (see [Threading](./threading.md)).

### Delivery Status Notifications

Request delivery status notifications (DSN):

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "With DSN",
  "text": "Request delivery notification",
  "dsn": {
    "return": "headers",
    "notify": ["success", "failure", "delay"]
  }
}
```

Note: DSN support varies by email provider.

### Email Tracking

Enable open and click tracking for your emails:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Tracked Email",
  "html": "<p>Check out <a href='https://example.com'>our website</a>!</p>",
  "trackOpens": true,
  "trackClicks": true
}
```

When enabled, EmailEngine will:

- Insert a tracking pixel to detect email opens (`trackOpens`)
- Rewrite links to track clicks (`trackClicks`)
- Send `trackOpen` and `trackClick` webhook events when detected

**Optional:** Override the base URL for tracking links:

```json
{
  "trackOpens": true,
  "trackClicks": true,
  "baseUrl": "https://yourdomain.com"
}
```

[Learn more about tracking events →](/docs/webhooks/trackopen)

### Preview Mode (Dry Run)

Generate email preview without actually sending:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Test Email",
  "html": "<p>Preview this email</p>",
  "dryRun": true
}
```

**Response:**

```json
{
  "response": "Dry run",
  "messageId": "<generated-message-id@example.com>",
  "preview": "BASE64_ENCODED_RFC822_MESSAGE"
}
```

The `preview` field contains the complete RFC822 formatted email (base64 encoded). Decode it to see exactly what would be sent. Perfect for testing templates and rendering.

### Network Configuration

#### Proxy Routing

Route SMTP connection through a proxy server:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Via Proxy",
  "text": "Sent through proxy",
  "proxy": "http://proxy.company.com:8080"
}
```

Supports HTTP, HTTPS, SOCKS4, and SOCKS5 proxies.

#### Local Address Binding

Bind to a specific local IP address:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "From Specific IP",
  "text": "Sent from specific interface",
  "localAddress": "192.168.1.100"
}
```

Useful for multi-interface systems or IP-based routing.

### Delivery Control

#### Custom Retry Attempts

Override the default retry count for this message:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "High Priority",
  "text": "Will retry up to 15 times",
  "deliveryAttempts": 15
}
```

Default is usually 10 attempts. Use higher values for critical emails.

#### SMTP Gateway

Route through a specific SMTP gateway:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Via Gateway",
  "text": "Sent through custom gateway",
  "gateway": "gateway-id-123"
}
```

Gateways are SMTP accounts (like SendGrid, Mailgun, or Amazon SES) that EmailEngine can use to send emails on behalf of any account. Register gateways via the [Gateway API](/docs/api/post-v-1-gateway).

#### SMTP Envelope

Specify SMTP envelope separately from message headers:

```json
{
  "to": [{ "address": "recipient@example.com" }],
  "from": { "address": "noreply@example.com" },
  "subject": "Envelope Example",
  "text": "Header From and SMTP MAIL FROM can differ",
  "envelope": {
    "from": "bounce@example.com",
    "to": ["actualrecipient@example.com"]
  }
}
```

Useful for bounce handling and advanced email routing.

### Idempotency

Prevent duplicate message submission with idempotency keys:

**Request:**

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-key-12345" \
  -d '{
    "to": [{ "address": "recipient@example.com" }],
    "subject": "Important",
    "text": "This will only send once even if request is retried"
  }'
```

If the same request is sent multiple times with the same `Idempotency-Key` header, EmailEngine will:

- Process it only once
- Return the same response for duplicate requests
- Prevent accidental double-sends

The idempotency key can be any string (0-1024 characters). Use UUIDs or request-specific identifiers.

## Sending Stored Drafts

Send a draft that already exists in the mailbox by referencing its message ID:

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/example/message/AAAAAQAACnA/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

The referenced message must be a draft: stored in the Drafts folder or flagged as `\Draft` for IMAP accounts, carrying the `DRAFT` label on Gmail, or a draft message on MS Graph. Drafts can come from the user's mail client, or be created through the [message upload API](/docs/api/post-v-1-account-account-message) by uploading to the Drafts folder with the `\Draft` flag.

The submission is queued like any other email, so scheduled sending, retry handling, gateways, idempotency keys, and the delivery webhooks described below all behave the same as with `/submit`. The response contains the familiar `queueId` and `messageId` values.

What happens to the draft after sending depends on the account type:

- **Gmail and MS Graph accounts** use the provider's native draft-send call. The provider files the message into the Sent Mail folder and removes the draft.
- **IMAP accounts** download the draft and deliver it over SMTP. A copy is stored in the Sent Mail folder (unless disabled with `copy: false` or the mail server stores sent messages itself, as Gmail and Outlook do for SMTP submissions), and the draft is then deleted. If no sent copy exists anywhere, the draft is moved to Trash instead so its content is not lost.

The request body is optional and accepts the delivery options described above - `envelope`, `copy`, `sentMailPath`, `sendAt`, `deliveryAttempts`, `gateway`, `dsn`, `proxy`, and `localAddress`. Content fields are not accepted; the draft is sent as stored. See the [draft submission API reference](/docs/api/post-v-1-account-account-message-message-submit) for the full schema.

## Webhook Notifications

EmailEngine sends webhook notifications for delivery status updates. Configure your webhook URL under **Settings → Webhooks**.

### messageSent

Delivered to the outbound MTA (SMTP server accepted the message):

```json
{
  "account": "example",
  "date": "2025-05-14T10:32:39.499Z",
  "event": "messageSent",
  "data": {
    "messageId": "<a00576bd-f757-10c7-26b8-885d7bbd9e83@example.com>",
    "response": "250 2.0.0 Ok: queued as 5755482356",
    "envelope": {
      "from": "andris@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

### messageDeliveryError

Emitted **after every failed delivery attempt**. EmailEngine retries automatically until delivery succeeds or the maximum number of attempts is reached:

```json
{
  "serviceUrl": "http://127.0.0.1:3000",
  "account": "example",
  "date": "2025-05-14T15:07:35.832Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "1833c8a88a86109a1bf",
    "envelope": {
      "from": "andris@example.com",
      "to": ["recipient@example.com"]
    },
    "messageId": "<29e26263-7125-ff56-4f80-83a5cf737d5e@example.com>",
    "error": "400 Message Not Accepted",
    "errorCode": "EPROTOCOL",
    "smtpResponseCode": 400,
    "job": {
      "attemptsMade": 1,
      "attempts": 10,
      "nextAttempt": "2025-05-14T15:07:45.465Z"
    }
  }
}
```

### messageFailed

Raised once EmailEngine gives up retrying (max attempts reached):

```json
{
  "account": "example",
  "date": "2025-05-14T11:58:50.181Z",
  "event": "messageFailed",
  "data": {
    "messageId": "<97ac5d9a-93c7-104b-8d26-6b25f8d644ec@example.com>",
    "queueId": "610c2c93e608bd37",
    "error": "Error: Invalid login: 535 5.7.8 Error: authentication failed: "
  }
}
```

### Bounce Handling

A `messageSent` event means the account's email server (e.g., Gmail, Outlook) accepted the email for delivery. However, when that server's MTA attempts to deliver the email to the recipient's mail server (MX), the recipient MX may reject it (e.g., mailbox full, user doesn't exist, domain not found).

When this happens, the sender's MTA generates a bounce response email and sends it back to the sender's inbox. This is an informational email that explains why delivery failed. EmailEngine monitors the inbox, detects these bounce emails, parses them to extract the bounced recipient and error details, and triggers a `messageBounce` webhook event if it can identify which original message bounced.

See:

- [Bounce Detection](/docs/advanced/bounces) - How EmailEngine detects and processes bounces
- [messageBounce Webhook](/docs/webhooks/messagebounce) - Webhook payload and handling

## Testing Sent Emails

### Using Ethereal Email

For testing, use [Ethereal Email](https://ethereal.email/) to create temporary test accounts:

```bash
# Send to your Ethereal test address
curl -XPOST "http://127.0.0.1:3000/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{ "address": "test@ethereal.email" }],
    "subject": "Test Email",
    "text": "Testing EmailEngine sending"
  }'
```

### Check Sent Folder

Verify the email was saved to the Sent folder:

```bash
curl "http://127.0.0.1:3000/v1/account/example/messages?path=Sent" \
  -H "Authorization: Bearer <token>"
```

### Monitor Queue

Check the outbox queue status. The outbox is a single global queue (it is not scoped per account):

```bash
curl "http://127.0.0.1:3000/v1/outbox" \
  -H "Authorization: Bearer <token>"
```

## Common Pitfalls

### Authentication Issues

**Problem**: Gmail, Outlook, and Yahoo may refuse SMTP logins that look like bots.

**Solution**:

- **Gmail**: Use OAuth2 or app-specific passwords (account passwords no longer work)
- **Outlook**: Use OAuth2 (password authentication completely disabled)
- **Yahoo**: Use OAuth2 or app-specific passwords

See [Gmail Setup](../accounts/gmail/gmail-imap.md) and [Outlook Setup](../accounts/microsoft-365/outlook-365.md).

### Timeout Errors

**Problem**: Heroku dynos cut idle sockets.

**Solution**:

- Move off Heroku or increase dyno size
- Configure longer timeouts
- Use a different deployment platform

### Account Not Connected

**Problem**: Submission rejected with "Account not ready" error.

**Solution**: Wait for initial IMAP sync to complete. Poll `/v1/account/:id` until `state` becomes `"connected"`.

### Rate Limiting

**Problem**: Too many emails sent too quickly.

**Solution**:

- Implement throttling in your application
- Use mail merge for bulk sending
- Spread sends over time
- Check provider rate limits

### Large Attachments

**Problem**: Email rejected due to size limits.

**Solution**:

- Reduce attachment size
- Use external file hosting with links
- Compress attachments
- Check provider size limits (typically 25-50MB)

### Spam Filters

**Problem**: Emails flagged as spam.

**Solution**:

- Provide both HTML and plain text versions
- Avoid spam trigger words
- Include unsubscribe links for bulk email
- Authenticate domain with SPF/DKIM
- Warm up new accounts gradually

## Performance Considerations

### Optimize for Bulk Sending

For sending multiple emails:

- Use [Mail Merge](./mail-merge.md) instead of individual calls
- Enable `copy: false` to skip Sent folder storage
- Implement rate limiting
- Monitor the outbox queue

### Connection Pooling

EmailEngine maintains SMTP connection pools automatically. For high-volume sending:

- Keep accounts connected
- Avoid frequent reconnections
- Monitor connection status

### Webhook Processing

Handle webhooks asynchronously:

- Return 200 OK quickly from webhook endpoint
- Process delivery status in background jobs
- Implement webhook retry logic

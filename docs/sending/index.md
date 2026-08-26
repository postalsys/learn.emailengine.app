---
title: Sending Emails
sidebar_position: 1
description: Overview of EmailEngine's email sending capabilities including SMTP proxying, API submission, and mail merge
---

# Sending Emails

Send through a registered account's own SMTP server, or through an external sending service, with the same REST call either way.

## Why Use EmailEngine for Sending

When your application needs to send email on behalf of users, direct SMTP integration is complex and brittle:

- **Provider diversity**: Every email provider has different authentication mechanisms, rate limits, and error codes
- **Credential management**: Securely handling user SMTP credentials is challenging
- **Retry logic**: Deciding which failures are worth retrying, and when
- **Queue management**: Handling message queues and delivery tracking
- **OAuth complexity**: Modern providers require OAuth2 authentication

EmailEngine puts one REST endpoint in front of all of it, with the same request and response shape whichever provider is behind the account.

## Key Capabilities

### Unified API
- Single REST endpoint (`/v1/account/{account}/submit`) for all providers
- Consistent JSON request/response format
- Automatic credential management

### Reliable Delivery
- Built-in message queuing
- Automatic retry logic with exponential backoff
- Delivery status tracking via webhooks
- SMTP connection pooling

### Advanced Features
- Mail merge for bulk personalized emails
- Email templates with Handlebars
- Proper reply and forward threading
- Attachment handling
- Custom headers and MIME options

### Flexible Sending Methods

EmailEngine supports multiple sending approaches:

1. **Submit API** (Recommended)
   - POST to [Submit Email API endpoint](/docs/api/post-v-1-account-account-submit)
   - Queue-based with automatic retries
   - Webhook notifications for delivery status
   - Also sends [stored drafts](./basic-sending.md#sending-stored-drafts) by message ID
   - Best for application integration

2. **SMTP Server**
   - Direct SMTP server provided by EmailEngine (its own MSA)
   - Use standard SMTP clients/libraries
   - EmailEngine routes to the correct account based on the authenticated account ID
   - Same queue and webhooks as the submit API; mail merge, templates, and replies are not available this way
   - Best for legacy applications

   Not to be confused with an SMTP **Gateway**, which is a separate feature - an outbound relay account (such as Amazon SES or a corporate smarthost) that you select per message via the `gateway` field.

## Quick Examples

### Simple Email

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": {
      "name": "Recipient Name",
      "address": "recipient@example.com"
    },
    "subject": "Hello from EmailEngine",
    "text": "Plain text version",
    "html": "<p>HTML version</p>"
  }'
```

### Reply to Email

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": {
      "message": "AAAADQAABl0",
      "action": "reply"
    },
    "html": "<p>Your reply content</p>"
  }'
```

### Send a Stored Draft

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/message/AAAADQAABl0/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

### Mail Merge

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Hello {{params.name}}",
    "html": "<p>Personal message for {{params.name}}</p>",
    "mailMerge": [
      {
        "to": {"address": "alice@example.com"},
        "params": {"name": "Alice"}
      },
      {
        "to": {"address": "bob@example.com"},
        "params": {"name": "Bob"}
      }
    ]
  }'
```

## Understanding the Send Process

When you submit an email through EmailEngine:

1. **Validation**: EmailEngine validates your request payload
2. **Queuing**: Message is added to the outbox queue with a unique queue ID
3. **Processing**: Message is picked up from the queue for delivery
4. **SMTP Transfer**: EmailEngine connects to the account's SMTP server
5. **Notification**: Webhooks notify you of delivery status
6. **Storage**: Copy saved to Sent Mail folder (optional)

## Delivery Status Tracking

EmailEngine sends webhook notifications for every stage:

- **Queued**: Message accepted and queued (`response` in submit API)
- **Sending**: Message being transmitted (no webhook)
- **Sent**: Delivered to SMTP server (`messageSent` webhook)
- **Retry**: Temporary failure, will retry (`messageDeliveryError` webhook)
- **Failed**: Permanent failure after retries (`messageFailed` webhook)

## Common Use Cases

### Transactional Emails
Send receipts, confirmations, and notifications from user mailboxes:
- Order confirmations
- Password reset emails
- Account notifications

### Support Communication
Handle customer support emails:
- Reply to support tickets
- Forward emails to team members
- Maintain conversation threads

### Marketing & Outreach
Personalized bulk sending:
- Mail merge campaigns
- Newsletter distribution
- Follow-up sequences

### Automated Workflows
Integrate email into your application logic:
- Trigger emails from events
- Send scheduled reminders
- Process email templates

## Getting Started

1. **[Basic Sending](./basic-sending.md)** - Learn the fundamentals of sending emails
2. **[Replies & Forwards](./replies-forwards.md)** - Properly reply to and forward emails
3. **[Mail Merge](./mail-merge.md)** - Send bulk personalized emails
4. **[Threading](/docs/sending/threading)** - Maintain conversation threads
5. **[Templates](./templates.md)** - Use email templates
6. **[Outbox Queue](./outbox-queue.md)** - Understanding the queue system
7. **[SMTP Server](./smtp-interface.md)** - Alternative SMTP integration

## See Also

- [Outbox queue](/docs/sending/outbox-queue) - What happens between "queued" and "sent"
- [messageSent](/docs/webhooks/messagesent) and [messageFailed](/docs/webhooks/messagefailed) - The delivery events to handle
- [Sending API](/docs/api-reference/sending-api) - Every field the submit endpoint accepts
- [Bounces](/docs/advanced/bounces) - Recognizing a delivery failure that arrives by mail rather than by webhook
- [SMTP gateways](/docs/sending/transactional-service) - Relaying through a sending service instead of the user's own server

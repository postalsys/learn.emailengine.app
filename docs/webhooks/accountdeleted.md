---
title: "accountDeleted"
sidebar_position: 17
description: "Webhook event triggered when an email account is removed from EmailEngine"
---

# accountDeleted

The `accountDeleted` webhook event is triggered when an email account is removed from EmailEngine. It is the final event in the account lifecycle and means the account configuration and its cached data have been deleted.

## When This Event is Triggered

The `accountDeleted` event fires when:

- An account is deleted via the [Delete Account API](/docs/api/delete-v-1-account-account)
- An account is removed through the admin interface

Both paths go through the same deletion routine. The account record, its stored credentials, mailbox listing and cached message index are removed from Redis first, then the main process sends this webhook and tells the worker that held the connection to tear it down.

This is the one event that is still delivered for an account that no longer exists. Every other webhook is dropped when its account cannot be found at delivery time.

## Common Use Cases

- **Account cleanup** - Remove associated data from your application database
- **Billing integration** - Stop billing cycles when accounts are removed
- **User notifications** - Inform users their email connection has been removed
- **Audit logging** - Track all account deletions for compliance purposes
- **Dashboard updates** - Remove deleted accounts from your UI
- **Resource cleanup** - Release any resources allocated for the account
- **Analytics tracking** - Monitor account churn and retention metrics

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | The account ID of the deleted account |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `accountDeleted` |
| `data` | object | Yes | Event data object |

### Event Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account` | string | Yes | The account ID, the same value as the top-level `account` field |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header, which is what to deduplicate on. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T14:32:45.892Z",
  "event": "accountDeleted",
  "data": {
    "account": "user123"
  }
}
```

## Example Payload (Without Service URL)

When no service URL is configured:

```json
{
  "serviceUrl": null,
  "account": "gmail-user456",
  "date": "2025-10-17T16:45:00.000Z",
  "event": "accountDeleted",
  "data": {
    "account": "gmail-user456"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleAccountDeleted(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  console.log(`Account deleted: ${account}`);
  console.log(`  Time: ${date}`);
  console.log(`  Event ID: ${eventId}`);

  // Remove account from your system
  await deleteAccountRecord(account);
}
```

### Cleaning Up Account Data

```javascript
async function handleAccountDeleted(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  // Delete account record from database
  await db.accounts.delete({
    where: { emailEngineId: account }
  });

  // Clean up related data
  await db.messages.deleteMany({
    where: { accountId: account }
  });

  await db.contacts.deleteMany({
    where: { accountId: account }
  });

  // Log the account deletion
  await auditLog.create({
    event: 'account_deleted',
    account,
    timestamp: date,
    eventId
  });
}
```

### Billing Integration

```javascript
async function handleAccountDeleted(event) {
  const { account, date } = event;

  // Get user associated with this account
  const accountRecord = await db.accounts.findUnique({
    where: { emailEngineId: account },
    include: { user: true }
  });

  if (accountRecord && accountRecord.user) {
    // Update billing records
    await billing.removeAccount({
      userId: accountRecord.user.id,
      accountId: account,
      endDate: new Date(date)
    });

    // Recalculate subscription if needed
    const remainingAccounts = await getAccountCount(accountRecord.user.id);
    if (remainingAccounts === 0) {
      await billing.pauseSubscription(accountRecord.user.id);
    }
  }

  // Now safe to delete the account record
  await db.accounts.delete({
    where: { emailEngineId: account }
  });
}
```

### User Notifications

```javascript
async function handleAccountDeleted(event) {
  const { account, date } = event;

  // Look up user before deleting account record
  const accountRecord = await db.accounts.findUnique({
    where: { emailEngineId: account },
    include: { user: true }
  });

  if (accountRecord && accountRecord.user) {
    // Notify user about account removal
    await sendNotification({
      userId: accountRecord.user.id,
      type: 'account_removed',
      message: 'Your email account has been disconnected.',
      metadata: {
        accountId: account,
        removedAt: date
      }
    });
  }

  // Clean up account data
  await db.accounts.delete({
    where: { emailEngineId: account }
  });
}
```

### Idempotent Handler

A failed or timed-out delivery is retried, so the same event can arrive more than once. Deduplicate on the event ID header:

```javascript
async function handleAccountDeleted(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  // Check if we've already processed this event
  const existingEvent = await db.processedEvents.findUnique({
    where: { eventId }
  });

  if (existingEvent) {
    console.log(`Event ${eventId} already processed, skipping`);
    return;
  }

  // Record that we're processing this event
  await db.processedEvents.create({
    data: {
      eventId,
      eventType: 'accountDeleted',
      processedAt: new Date()
    }
  });

  // Check if account still exists in our system
  const accountRecord = await db.accounts.findUnique({
    where: { emailEngineId: account }
  });

  if (!accountRecord) {
    console.log(`Account ${account} already deleted, skipping`);
    return;
  }

  // Proceed with deletion
  await db.accounts.delete({
    where: { emailEngineId: account }
  });

  await auditLog.create({
    event: 'account_deleted',
    account,
    timestamp: date,
    eventId
  });
}
```

## Event Sequence

When an account is deleted, you receive only:

1. **`accountDeleted`** - Account is removed (this event)

No other events follow, because the account no longer exists. Events for the same account that were still queued when it was deleted are dropped rather than delivered.

### Complete Account Lifecycle

A typical complete account lifecycle includes:

1. **`accountAdded`** - Account is registered
2. **`authenticationSuccess`** - Account successfully authenticates
3. **`accountInitialized`** - Initial mailbox sync is complete
4. *(Account is active, various message and mailbox events may occur)*
5. **`accountDeleted`** - Account is removed (final event)

## Important Considerations

### Data Already Deleted

When you receive the `accountDeleted` webhook, the account data has already been removed from EmailEngine. [Get Account](/docs/api/get-v-1-account-account) returns a 404 for it. If you need account metadata for cleanup purposes, store it in your own database when the account is created.

### Timing

The webhook is queued as part of the deletion, so it arrives as soon as delivery succeeds. There is no delay between the deletion and the notification other than queue and network time.

## Related Events

- [accountAdded](/docs/webhooks/accountadded) - Triggered when a new account is registered
- [accountInitialized](/docs/webhooks/accountinitialized) - Triggered when initial sync completes
- [authenticationSuccess](/docs/webhooks/authenticationsuccess) - Triggered when authentication succeeds
- [authenticationError](/docs/webhooks/authenticationerror) - Triggered when authentication fails
- [connectError](/docs/webhooks/connecterror) - Triggered when connection fails

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries and the `webhookEvents` allowlist
- [Account Management](/docs/accounts/managing-accounts) - Deleting accounts and what deletion removes
- [Delete Account API](/docs/api/delete-v-1-account-account) - The request that triggers this event
- [Get Account API](/docs/api/get-v-1-account-account) - What to store before the account is gone

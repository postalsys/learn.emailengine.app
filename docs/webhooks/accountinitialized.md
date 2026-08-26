---
title: "accountInitialized"
sidebar_position: 16
description: "Webhook event triggered when an email account completes its initial mailbox synchronization"
---

# accountInitialized

The `accountInitialized` webhook event is triggered when an email account reaches the `connected` state for the first time. For an IMAP account that is after the first pass over its folders; for a Gmail API or Microsoft Graph account it is after the provider profile check succeeds. From this point the account is operational.

## When This Event is Triggered

The `accountInitialized` event fires when:

- An account reaches the `connected` state for the **first time** after being added
- An account reaches the `connected` state again after a [flush](/docs/api/put-v-1-account-account-flush)

It fires once per initialization cycle. Routine reconnections, restarts and recoveries from error states do not fire it again unless the account has been flushed in between.

### Technical Details

EmailEngine keeps a per-account counter of how many times the account has entered the `connected` state. The counter is created at `0` when the account is registered, and reset to `0` by a flush. When the state becomes `connected` and the counter moves from `0` to `1`, the event is sent.

## Common Use Cases

- **Account activation confirmation** - Know when accounts are ready to use
- **Onboarding completion** - Mark user onboarding as complete when their email is connected
- **Initial data sync** - Trigger processes that need mailbox data to be available
- **User notifications** - Inform users their email account is now active
- **Dashboard updates** - Update account status to "active" or "ready"
- **Start message processing** - Begin automated email processing workflows

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | The account ID that was initialized |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `accountInitialized` |
| `data` | object | Yes | Event data object |

### Event Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `initialized` | boolean | Yes | Always `true` |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header, which is what to deduplicate on. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:50:45.321Z",
  "event": "accountInitialized",
  "data": {
    "initialized": true
  }
}
```

## Example Payload (Without Service URL)

When no service URL is configured:

```json
{
  "serviceUrl": null,
  "account": "gmail-user456",
  "date": "2025-10-17T08:16:15.000Z",
  "event": "accountInitialized",
  "data": {
    "initialized": true
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleAccountInitialized(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  console.log(`Account initialized: ${account}`);
  console.log(`  Time: ${date}`);
  console.log(`  Event ID: ${eventId}`);

  // Mark account as ready in your system
  await markAccountReady(account);
}
```

### Updating Account Status

```javascript
async function handleAccountInitialized(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  // Update account status to active
  await db.accounts.update({
    where: { emailEngineId: account },
    data: {
      status: 'active',
      initializedAt: new Date(date),
      lastEventId: eventId
    }
  });

  // Log the initialization
  await auditLog.create({
    event: 'account_initialized',
    account,
    timestamp: date,
    eventId
  });
}
```

### Completing User Onboarding

```javascript
async function handleAccountInitialized(event) {
  const { account, date } = event;

  // Update account status
  await db.accounts.update({
    where: { emailEngineId: account },
    data: {
      status: 'active',
      initializedAt: new Date(date)
    }
  });

  // Notify user that setup is complete
  const user = await getUserByAccount(account);
  if (user) {
    await sendNotification({
      userId: user.id,
      type: 'account_ready',
      message: 'Your email account is now connected and ready to use'
    });

    // Complete onboarding if this was their first account
    if (!user.onboardingComplete) {
      await completeOnboarding(user.id);
    }
  }
}
```

### Starting Automated Processing

```javascript
async function handleAccountInitialized(event) {
  const { account } = event;

  // Mark account as ready
  await db.accounts.update({
    where: { emailEngineId: account },
    data: { status: 'active' }
  });

  // Start any automated email processing for this account
  await startEmailProcessor(account);

  // Fetch initial mailbox statistics
  const mailboxes = await emailEngine.getMailboxes(account);
  await cacheMailboxStats(account, mailboxes);
}
```

## Event Sequence

When a new IMAP account is added, webhooks arrive in this order:

1. **`accountAdded`** - Account configuration is stored
2. **`authenticationSuccess`** - The mail server accepted the login
3. **`accountInitialized`** - The first pass over the folders is complete (this event)

For a Gmail API or Microsoft Graph account, `accountInitialized` is sent before `authenticationSuccess`, because the state becomes `connected` as soon as the provider profile check succeeds and the success notification follows it. Do not depend on the order between the two.

### Re-initialization After Flush

The [Flush Account API](/docs/api/put-v-1-account-account-flush) resets the connection counter to `0` and discards the account's mailbox listing and sync state, so:

1. The account disconnects and re-syncs from the current point in time
2. A new `accountInitialized` event fires when it reaches `connected` again

Flushing is the way to re-run the initial sync after a configuration change without deleting and re-adding the account.

## Differences from Other Account Events

| Event | When Triggered | What It Means |
|-------|----------------|---------------|
| `accountAdded` | After account creation | Account config is stored, connection not yet attempted |
| `authenticationSuccess` | After successful authentication | Account can connect to mail server |
| `accountInitialized` | On the first `connected` state | Account is operational (this event) |
| `accountDeleted` | When account is removed | Account has been deleted from EmailEngine |

## Related Events

- [accountAdded](/docs/webhooks/accountadded) - Triggered when account is first registered
- [authenticationSuccess](/docs/webhooks/authenticationsuccess) - Triggered when authentication succeeds
- [authenticationError](/docs/webhooks/authenticationerror) - Triggered when authentication fails
- [connectError](/docs/webhooks/connecterror) - Triggered when the connection fails before authentication
- [accountDeleted](/docs/webhooks/accountdeleted) - Triggered when an account is removed

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Account Management](/docs/accounts/managing-accounts) - Account states and the lifecycle around them
- [Create Account API](/docs/api/post-v-1-account) - Registering the account this event follows
- [Flush Account API](/docs/api/put-v-1-account-account-flush) - Re-running the initial sync and this event

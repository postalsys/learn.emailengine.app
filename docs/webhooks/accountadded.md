---
title: "accountAdded"
sidebar_position: 15
description: "Webhook event triggered when a new email account is registered with EmailEngine"
---

# accountAdded

The `accountAdded` webhook event is triggered when a new email account is registered with EmailEngine. It is the first event in the account lifecycle and means the account configuration has been accepted and stored. Nothing has been connected yet.

## When This Event is Triggered

The `accountAdded` event fires once, when an account ID that EmailEngine has not seen before is registered:

- Through the [Create Account API](/docs/api/post-v-1-account)
- Through the [hosted authentication form](/docs/accounts/hosted-authentication)
- Through the admin interface

The event is sent by the main process after the account has been assigned to a worker and **before** that worker attempts a connection. Authentication has not been verified at this point.

Re-registering an account ID that already exists does not fire it again. A `POST /v1/account` for an existing account, or a hosted-form re-authorization of one, updates the stored account and reconnects it instead. The lifecycle events that follow a fresh registration are described under [Event Sequence](#event-sequence).

Like every event, it is delivered only if `accountAdded` or `*` is in `webhookEvents`. See [Webhooks Overview](/docs/webhooks/overview) for the allowlist.

## Common Use Cases

- **Account registration tracking** - Log when new accounts are added to your system
- **Onboarding workflows** - Create the local record that later lifecycle events update
- **Billing integration** - Start billing cycles when accounts are registered
- **User notifications** - Inform users their account is being set up
- **Audit logging** - Track all account additions for compliance purposes
- **Dashboard updates** - Show newly added accounts in a pending or connecting state

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | The account ID that was registered |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `accountAdded` |
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
  "date": "2025-10-17T06:49:22.157Z",
  "event": "accountAdded",
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
  "date": "2025-10-17T08:15:30.000Z",
  "event": "accountAdded",
  "data": {
    "account": "gmail-user456"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleAccountAdded(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  console.log(`New account registered: ${account}`);
  console.log(`  Time: ${date}`);
  console.log(`  Event ID: ${eventId}`);

  // Record the new account in your system
  await createAccountRecord(account);
}
```

### Creating Account Records in Database

```javascript
async function handleAccountAdded(event, headers) {
  const { account, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  // Create initial account record
  await db.accounts.create({
    data: {
      emailEngineId: account,
      status: 'connecting',
      createdAt: new Date(date),
      lastEventId: eventId
    }
  });

  // Log the account addition
  await auditLog.create({
    event: 'account_added',
    account,
    timestamp: date,
    eventId
  });
}
```

### Triggering Onboarding Workflows

```javascript
async function handleAccountAdded(event) {
  const { account, date } = event;

  // Create account record with pending status
  await db.accounts.create({
    data: {
      emailEngineId: account,
      status: 'pending_authentication',
      createdAt: new Date(date)
    }
  });

  // Send a notification to the user who owns the account
  const user = await getUserByAccount(account);
  if (user) {
    await sendNotification({
      userId: user.id,
      type: 'account_connecting',
      message: 'Your email account is being connected'
    });
  }

  // Check back if neither authenticationSuccess nor an error event has arrived
  await scheduleJob('check_account_connection', {
    account,
    checkAfterMinutes: 5
  });
}
```

### Billing Integration

```javascript
async function handleAccountAdded(event) {
  const { account, date } = event;

  // Get user associated with this account
  const user = await getUserByAccount(account);

  if (user) {
    // Update billing records
    await billing.addAccount({
      userId: user.id,
      accountId: account,
      startDate: new Date(date)
    });

    // Check account limits
    const accountCount = await getAccountCount(user.id);
    const plan = await getUserPlan(user.id);

    if (accountCount > plan.maxAccounts) {
      await billing.upgradeRequired(user.id, 'account_limit_exceeded');
    }
  }
}
```

## Event Sequence

When a new account is added and connects, you receive:

1. **`accountAdded`** - Account is registered (this event)
2. **`authenticationSuccess`** - The mail server or provider accepted the credentials
3. **`accountInitialized`** - The account reached the `connected` state for the first time

For an IMAP account the last two arrive in that order: `authenticationSuccess` is sent as soon as the login succeeds, and `accountInitialized` after the first pass over the folders. For a Gmail API or Microsoft Graph account both are sent during initialization and `accountInitialized` comes first. Do not depend on the order between them.

If authentication fails:

1. **`accountAdded`** - Account is registered (this event)
2. **`authenticationError`** or **`connectError`** - The credentials were rejected, or the server could not be reached

## Differences from Other Account Events

| Event | When Triggered | What It Means |
|-------|----------------|---------------|
| `accountAdded` | Immediately after account creation | Account config is stored, connection not yet attempted |
| `authenticationSuccess` | After successful authentication | Account can connect to mail server |
| `accountInitialized` | After the first successful sync | Mailboxes and messages are available |
| `accountDeleted` | When account is removed | Account has been deleted from EmailEngine |

## Related Events

- [authenticationSuccess](/docs/webhooks/authenticationsuccess) - Triggered when authentication succeeds
- [authenticationError](/docs/webhooks/authenticationerror) - Triggered when authentication fails
- [connectError](/docs/webhooks/connecterror) - Triggered when the connection fails before authentication
- [accountInitialized](/docs/webhooks/accountinitialized) - Triggered when the first sync completes
- [accountDeleted](/docs/webhooks/accountdeleted) - Triggered when an account is removed

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Account Management](/docs/accounts/managing-accounts) - Account states and what each one means
- [Create Account API](/docs/api/post-v-1-account) - The request that registers an account
- [Hosted Authentication](/docs/accounts/hosted-authentication) - Letting users register their own accounts through a form

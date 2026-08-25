---
title: Virtual Mailing Lists
sidebar_position: 10
description: Add unsubscribe headers, a hosted unsubscribe page, and automatic suppression to bulk sends without running a mailing list manager
keywords:
  - mailing lists
  - list-unsubscribe
  - unsubscribe
  - mail merge
  - suppression list
  - bulk email
---

# Virtual Mailing Lists

Add a `listId` to a [mail merge](/docs/sending/mail-merge) and EmailEngine treats the send as a mailing list. It adds one-click unsubscribe headers, hosts the unsubscribe page, records every opt-out, and skips those addresses on later sends to the same list.

Nothing needs to be registered first. A list ID you have not used before defines a new list.

:::tip The whole feature is one field

```json
{ "listId": "weekly-newsletter", "mailMerge": [{ "to": { "address": "subscriber@example.com" } }] }
```

Everything on this page describes what that field turns on.
:::

## Division of labor

Virtual lists cover the unsubscribe half of bulk sending. Your application still owns the list itself.

| EmailEngine handles | Your application handles |
| --- | --- |
| `List-ID`, `List-Unsubscribe` and `List-Unsubscribe-Post` headers | Who is on the list: signup, consent, storage |
| A signed unsubscribe URL for each recipient | Segmentation, scheduling and send volume |
| The hosted unsubscribe and re-subscribe page | Required footer content such as a postal address |
| Storing opt-outs and skipping those recipients on later sends | Reporting beyond EmailEngine's delivery, open and click tracking |
| `listUnsubscribe` and `listSubscribe` webhooks | |

EmailEngine never stores your subscribers, only the addresses that opted out.

## Requirements

### Set the service URL

The unsubscribe link points back at your EmailEngine instance, so the entire feature depends on the `serviceUrl` setting. Set it under Configuration - Service in the admin interface, or over the API:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"serviceUrl": "https://emailengine.example.com"}'
```

:::warning Without a service URL, listId does nothing

No headers are added and no suppression check runs, yet the submission still reports success. Recipients who already unsubscribed would receive the message. There is no error to catch, so confirm the setting before your first campaign.
:::

### Use mail merge

`listId` is only accepted alongside `mailMerge`, and the API rejects it on a plain submission. A merge with a single entry is fine when you want list handling for one recipient.

### Format the list ID as a hostname

Valid: `newsletter`, `weekly-updates`, `campaign-2026`. Invalid: `my_list` (underscore), `My List` (space), `list@example.com` (at sign).

## Send a campaign

```bash
curl -X POST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "listId": "weekly-newsletter",
    "subject": "This week at Example",
    "html": "<p>Hello {{name}},</p><p>...</p><p><a href=\"{{rcpt.unsubscribeUrl}}\">Unsubscribe</a></p>",
    "mailMerge": [
      { "to": { "address": "alice@example.com", "name": "Alice" }, "params": { "name": "Alice" } },
      { "to": { "address": "bob@example.com", "name": "Bob" }, "params": { "name": "Bob" } }
    ]
  }'
```

### Read the response

```json
{
  "mailMerge": [
    {
      "success": true,
      "to": { "address": "alice@example.com", "name": "Alice" },
      "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
      "queueId": "d41f0423195f271f"
    },
    {
      "success": true,
      "to": { "address": "bob@example.com", "name": "Bob" },
      "skipped": { "reason": "unsubscribe", "listId": "weekly-newsletter" }
    }
  ]
}
```

Suppressed recipients come back with `success: true` and a `skipped` object instead of a `messageId` and `queueId`. Nothing was queued for them, and this is not an error. Count the skipped entries if you want to track how much of a list has opted out.

## What the message carries

```
List-ID: <weekly-newsletter.emailengine.example.com>
List-Unsubscribe: <https://emailengine.example.com/unsubscribe?data=eyJhY3Qi...&sig=Ah0z...>
List-Unsubscribe-Post: List-Unsubscribe=One-Click
```

- `List-ID` combines your list ID with the hostname from `serviceUrl`.
- `List-Unsubscribe` is a signed URL unique to this recipient and this list. It answers both as a page (GET) and as a one-click target (POST).
- `List-Unsubscribe-Post` marks the link as RFC 8058 one-click, which is what Gmail and Yahoo require from bulk senders.

EmailEngine also generates a `Message-ID` when the submission does not provide one, so every opt-out can be traced back to the message that caused it.

### The recipient context

Setting a `listId` turns on Handlebars rendering for `subject`, `text`, `html` and `previewText`, even for entries with no `params`. Alongside your own merge parameters, each message gets an `rcpt` object:

| Variable | Value |
| --- | --- |
| `{{rcpt.unsubscribeUrl}}` | Signed unsubscribe URL for this recipient |
| `{{rcpt.address}}` | Recipient email address |
| `{{rcpt.name}}` | Recipient display name, when the merge entry supplies one |

Put the unsubscribe link in the message body as well as the header, since not every mail client surfaces the header version. Click tracking deliberately leaves this URL alone, so the link a recipient clicks is always the real one.

Because rendering is always on for list sends, literal `{{` in your content needs escaping. See [Template escaping](/docs/sending/mail-merge#template-escaping).

## What the recipient sees

There are two routes out, and both end in the same place:

- **The mail client's own unsubscribe button.** Gmail, Apple Mail, Outlook.com and others show it because of the one-click headers. The client posts to EmailEngine directly and the recipient never leaves their inbox.
- **The link in the message body.** Opens the hosted page, where the recipient confirms.

The hosted page has three states: confirm the unsubscribe, unsubscribed with an offer to re-subscribe, and subscription resumed. A mistaken unsubscribe is therefore self-service to undo, with no support ticket and no work on your side.

The page is localized and follows the recipient's browser language, falling back to your configured default locale. Style it under Configuration - Branding, where the brand name, a custom header block and custom `<head>` markup apply to every public page.

## React to opt-outs

| Webhook | Fires when |
| --- | --- |
| [`listUnsubscribe`](/docs/webhooks/listunsubscribe) | An address is added to the list, through one-click or the hosted page |
| [`listSubscribe`](/docs/webhooks/listsubscribe) | A recipient re-subscribes on the hosted page |

Both carry the list ID, the recipient, the originating `messageId`, and the IP address and user agent behind the request.

```javascript
app.post('/webhooks/emailengine', (req, res) => {
  res.json({ success: true }); // acknowledge first, then process

  const { event, data } = req.body;

  if (event === 'listUnsubscribe') {
    subscribers.setStatus(data.recipient, data.listId, 'unsubscribed');
  } else if (event === 'listSubscribe') {
    subscribers.setStatus(data.recipient, data.listId, 'subscribed');
  }
});
```

Two behaviors are worth knowing when you write that handler:

- **Events fire only on an actual change.** Unsubscribing an already suppressed address is a silent no-op, so a mail client that retries its one-click request will not produce duplicate events. Removing an address through the API does not fire `listSubscribe` either, because only a recipient's own re-subscribe counts as one.
- **The suppression is stored before the webhook goes out.** A failed delivery is logged, not retried. Treat webhooks as a signal to update your own records rather than as the record itself. EmailEngine's list is the authority and you can read it back at any time.

## Manage the list

Suppression Lists in the admin interface sidebar shows every list that exists, with the number of addresses on each. A list appears here the moment its first address is suppressed.

![Suppression Lists page listing each list with its address count](/img/screenshots/suppression-lists.png)

Opening a list shows what was recorded for each address: why it was suppressed, how it got there, the account that was sending, and when. The `one-click` source below is the mail client unsubscribe button, and `api` is an address the application suppressed itself.

![Entries on a suppression list, with reason, source, account and date columns](/img/screenshots/suppression-list-entries.png)

From these pages you can add or remove addresses, open a recipient's subscription page, and delete a list outright. The same operations are available over the API:

| Operation | Endpoint |
| --- | --- |
| All lists with entry counts | [`GET /v1/blocklists`](/docs/api/get-v-1-blocklists) |
| Entries on one list | [`GET /v1/blocklist/{listId}`](/docs/api/get-v-1-blocklist-listid) |
| Suppress an address | [`POST /v1/blocklist/{listId}`](/docs/api/post-v-1-blocklist-listid) |
| Release an address | [`DELETE /v1/blocklist/{listId}`](/docs/api/delete-v-1-blocklist-listid) |

The API calls it a blocklist, but it is the same store the unsubscribe flow writes to. Adding addresses yourself is how you feed hard bounces and spam complaints into a list. See [Blocklist Management](/docs/advanced/blocklists) for entry metadata and examples.

Removing the last address on a list deletes the list itself. A later send with that list ID re-creates it.

## Limits to plan around

- **One list per submission.** A merge is checked against a single `listId`. To honor several suppression sources, consolidate them into one list or filter in your own application before submitting.
- **Suppression is per list.** Opting out of `product-updates` has no effect on `weekly-newsletter`. That is what makes granular preferences possible, and it also means a global opt-out is your application's job.
- **Unsubscribe links do not expire.** They stay valid as long as the service secret does. Rotating or losing that secret breaks the links in mail that has already been delivered.
- **Addresses are stored lowercased and trimmed,** so matching is case-insensitive.
- **No campaign tooling.** There is no contact storage, segmentation, A/B testing or list analytics. Virtual lists suit an application that already has those and needs compliant unsubscribe handling.

## See Also

- [Mail Merge](/docs/sending/mail-merge) - Personalized bulk sending, the delivery mechanism behind virtual lists
- [Blocklist Management](/docs/advanced/blocklists) - The suppression store, its API and entry metadata
- [Bounce Detection](/docs/advanced/bounces) - Feeding hard bounces into a list
- [listUnsubscribe](/docs/webhooks/listunsubscribe) and [listSubscribe](/docs/webhooks/listsubscribe) - Full webhook payloads

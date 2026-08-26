---
title: Provider-Specific Threading
sidebar_position: 2
description: Which backends give EmailEngine a native thread ID, what it looks like, and which of them can search a whole thread in one request
---

# Provider-Specific Threading Support

Whether a message carries a `threadId`, and whether a whole thread can be fetched in one request, depends on the backend the account uses. This page lists what each one provides.

## Gmail / Google Workspace

Both Gmail backends assign a thread ID to every message.

### Native Thread Support

**Backends**:

- IMAP with OAuth2: `threadId` from the `X-GM-THRID` attribute of Gmail's `X-GM-EXT-1` IMAP extension
- Gmail API: `threadId` from the API

**Characteristics**:

- Thread ID format: long numeric string, for example `"1759349012996310407"`, the same value over either backend
- Availability: webhooks, message listings, message details, search results
- The `\All` folder is available for cross-folder thread search
- On the Gmail API, `reference.threadId` on submit attaches an outgoing message to a thread directly; see [Sending threaded messages](/docs/sending/threading/sending-threaded#using-the-reference-api)

### Example Response

```bash
curl "https://emailengine.example.com/v1/account/gmail/messages?path=INBOX" \
  -H "Authorization: Bearer <token>"
```

```json
{
  "messages": [
    {
      "id": "AAABkPHBeR0",
      "threadId": "1759349012996310407",
      "subject": "Project discussion",
      "from": {
        "address": "colleague@example.com"
      }
    }
  ]
}
```

## Microsoft 365 / Outlook

Which backend the account uses decides everything here.

### Native Thread Support

**Backends**:

- Microsoft Graph API: `threadId` is the message's Graph `conversationId`
- IMAP with OAuth2: no thread ID

Only an account added with the **Microsoft Graph API** backend has a `threadId`. A Microsoft 365 account added over **IMAP with OAuth2** talks to Microsoft's IMAP server, which implements neither `OBJECTID` nor a Gmail-style extension, so it behaves like any other IMAP account: threads have to be built from headers.

**Characteristics (Graph API)**:

- Thread ID format: Graph conversation ID, for example `"AAQkAGI2THY2ZjRhLTVjNzgtNDMxYS05YTBmLTJiN2M4ZDkxZTQyMwAQAF3xTx0nRUxOhKcvLZQ9r1M="`
- Availability: webhooks, message listings, message details, search results
- The `\All` folder is available for cross-folder thread search; a `threadId` search on it filters on `conversationId`

### Example Response

```json
{
  "messages": [
    {
      "id": "AAMkAGI2THY2ZjRhLTVjNzgtNDMxYS05YTBmLTJiN2M4ZDkxZTQyMwBGAAAAAABBUq8CzXaBTKl9k7lJ0hJ7BwCXKmSHhLBFTr0AAA==",
      "threadId": "AAQkAGI2THY2ZjRhLTVjNzgtNDMxYS05YTBmLTJiN2M4ZDkxZTQyMwAQAF3xTx0nRUxOhKcvLZQ9r1M=",
      "subject": "Meeting tomorrow",
      "from": {
        "address": "manager@example.com"
      }
    }
  ]
}
```

**Backend comparison**:

| Backend             | Threading   | \All folder |
| ------------------- | ----------- | ----------- |
| Microsoft Graph API | Native      | Yes         |
| IMAP with OAuth2    | Manual only | No          |

## Yahoo / AOL / Verizon

Yahoo, AOL, and Verizon IMAP servers implement the `OBJECTID` extension (RFC 8474), and EmailEngine passes its `THREADID` attribute on as `threadId`.

**Characteristics**:

- Thread ID format: short numeric string, for example `"501"`
- Availability: webhooks, message listings, message details, search results
- **No `\All` folder**: search each folder separately and merge the results, as shown in [Searching threads](/docs/sending/threading/searching-threads#folder-by-folder-search)

### Example Response

```json
{
  "messages": [
    {
      "id": "AAAAKAAACKM",
      "threadId": "501",
      "subject": "Question about your service",
      "from": {
        "address": "customer@example.com"
      }
    }
  ]
}
```

The same applies to any other IMAP server that advertises `OBJECTID`: the thread ID is available, the `\All` folder is not.

## Other IMAP Providers

An IMAP server without `OBJECTID` or `X-GM-EXT-1` gives EmailEngine no thread identifier.

**Characteristics**:

- No `threadId` property in any response
- `threadId` cannot be used as a search term
- No `\All` folder
- Threads are reconstructed client-side from `Message-ID`, `In-Reply-To`, and `References`

The procedure is: search the relevant folders by `subject`, fetch the candidates, and group them on their headers. [Building threads manually](/docs/sending/threading/searching-threads#building-threads-manually) has the code.

## Provider Comparison Table

| Provider          | Backend       | Native threading | Thread ID format      | \All folder |
| ----------------- | ------------- | ---------------- | --------------------- | ----------- |
| Gmail             | IMAP + OAuth2 | Yes              | Long numeric          | Yes         |
| Gmail             | Gmail API     | Yes              | Long numeric          | Yes         |
| Microsoft 365     | Graph API     | Yes              | Graph conversation ID | Yes         |
| Microsoft 365     | IMAP + OAuth2 | No               | N/A (manual only)     | No          |
| Yahoo/AOL/Verizon | IMAP          | Yes (OBJECTID)   | Short numeric         | No          |
| Other IMAP        | IMAP          | No               | N/A (manual only)     | No          |

## Choosing the Backend

### Gmail accounts

Either backend gives you thread IDs and the `\All` folder. Choose between IMAP and the Gmail API on other grounds; see [Gmail accounts](/docs/accounts/gmail/gmail-imap).

### Microsoft 365 / Outlook accounts

Use the **Microsoft Graph API** backend if you need thread IDs or cross-folder thread search. Over IMAP, neither is available.

### Yahoo / AOL / Verizon

Standard IMAP. Thread IDs are available; plan for one search per folder.

### Other providers

Plan for client-side threading: building thread relationships from headers, searching several folders, and keeping any thread state in your own application.

## Migration Considerations

### Switching a Microsoft 365 account from IMAP to the Graph API

1. The account has to be re-authorized through an OAuth2 application with the Graph API base scopes; the IMAP grant does not carry over
2. Messages get new EmailEngine message IDs, and `threadId` becomes available
3. Update your application to use the Graph conversation IDs
4. Any thread mapping you built from headers stays valid only as long as you keep the headers it was built from

## Testing Thread Support

### Check the account type

```bash
curl "https://emailengine.example.com/v1/account/example" \
  -H "Authorization: Bearer <token>"
```

Two fields of the [account response](/docs/api/get-v-1-account-account) settle it together:

- `type` names the provider: `gmail` for a Google account, `outlook` for a Microsoft account, `imap` for a password-authenticated IMAP account
- `baseScopes` says which backend an OAuth2 account uses: `imap` for IMAP with OAuth2, `api` for the Gmail or Graph API. It is present only for OAuth2 accounts, and defaults to `imap` when the [OAuth2 application](/docs/api/get-v-1-oauth-2-app) does not set it

### Test thread ID availability

```bash
curl "https://emailengine.example.com/v1/account/example/messages?path=INBOX" \
  -H "Authorization: Bearer <token>"
```

If the entries have no `threadId`, the backend assigns none, and threads have to be built from headers.

## See Also

- [Threading overview](/docs/sending/threading/overview) - The headers and IDs this table is about
- [Searching threads](/docs/sending/threading/searching-threads) - The search strategy each provider needs
- [Account types](/docs/accounts) - Choosing the backend that gives you native threading
- [Searching messages](/docs/receiving/searching) - What else the search endpoint can filter on

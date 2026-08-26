---
title: Translations
sidebar_position: 14
description: Configure language settings for public-facing pages and contribute translations
---

# Translations

EmailEngine supports multiple languages for public-facing pages. This includes hosted authentication forms, error pages, unsubscribe pages, and other user-facing content.

:::note Admin Interface Not Translated
The admin dashboard and configuration interface are only available in English. Translations apply only to public pages that end users interact with.
:::

## Supported Languages

EmailEngine includes translations for the following languages:

| Language | Locale Code |
| -------- | ----------- |
| English  | `en`        |
| Estonian | `et`        |
| French   | `fr`        |
| German   | `de`        |
| Polish   | `pl`        |
| Japanese | `ja`        |
| Dutch    | `nl`        |

## Language Selection

EmailEngine determines the language to use based on the following priority order:

1. **Query parameter** - `?locale=fr`
2. **Custom header** - `X-EE-Locale: fr`
3. **Session cookie** - Stored from previous query/header selection
4. **Accept-Language header** - Browser's language preference
5. **Default locale** - Server-wide setting

### Per-Request Language (Query Parameter)

Append `locale=<code>` to the query string of any public page URL, for example a hosted authentication form URL returned by `POST /v1/authentication/form`:

```text
https://emailengine.example.com/accounts/new?data=eyJhY2NvdW50Ijoi...&sig=Ah0z...&locale=fr
```

When set via query parameter, the language selection is stored in a session cookie and persists until the browser session ends or a different language is selected.

### Per-Request Language (Header)

Set the `X-EE-Locale` header in your request. Here `FORM_URL` is the `url` returned by `POST /v1/authentication/form`:

```bash
curl "$FORM_URL" \
  -H "X-EE-Locale: de"
```

Like the query parameter, this also sets a session cookie to persist the selection.

### Browser Language Detection

If no explicit locale is set, EmailEngine uses the browser's `Accept-Language` header to negotiate the best available language. For example, a browser sending `Accept-Language: de-DE,de;q=0.9,en;q=0.8` would see German if available.

### Default Language (Server-Wide)

Set the server-wide default locale that applies when no other language preference is detected.

**Via API:**

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "locale": "fr"
  }'
```

**Via Web Interface:**

1. Open the EmailEngine admin dashboard
2. Navigate to **Configuration** > **General**
3. Find the **Default Language** setting
4. Select your preferred language from the dropdown
5. Click **Save**

### Pre-selecting Language in Hosted Authentication

When generating authentication form URLs, you can include the locale parameter to display the form in a specific language:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "redirectUrl": "https://myapp.com/settings"
  }'
```

Then append `&locale=fr` to the returned `url` before redirecting the user:

```text
https://emailengine.example.com/accounts/new?data=eyJhY2NvdW50Ijoi...&sig=Ah0z...&locale=fr
```

## What Gets Translated

### Public UI Pages

Translations apply to public-facing pages:

**Hosted Authentication Forms:**

- Account type selection ("Choose your email account provider")
- IMAP/SMTP configuration form labels and buttons ("Verify connection", "Save and continue")
- Connection test results ("Couldn't connect to IMAP server", "Server response:")
- Expired or invalid setup links ("Invalid or expired account setup URL")

**Unsubscribe Pages:**

- Unsubscribe confirmation
- Re-subscribe option
- Status messages

**Error Pages:**

- OAuth2 failures ("OAuth2 authentication failed") and missing-scope explanations
- Connection errors ("Could not connect to server")

**Redirect Pages:**

- "Click here to continue" messages

### API Validation Errors

API validation error messages are also translated. The same language selection mechanism applies to API requests:

**Triggers for API response translation:**

| Method                 | Example                      | Persists |
| ---------------------- | ---------------------------- | -------- |
| Query parameter        | `POST /v1/account?locale=fr` | No       |
| Custom header          | `X-EE-Locale: fr`            | No       |
| Accept-Language header | `Accept-Language: fr`        | No       |

**Example - German validation error:**

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-EE-Locale: de" \
  -H "Content-Type: application/json" \
  -d '{}'
```

Response with German error messages:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Ungültige Eingabe",
  "fields": [
    {
      "message": "\"account\" ist erforderlich",
      "key": "account"
    },
    {
      "message": "\"name\" ist erforderlich",
      "key": "name"
    }
  ]
}
```

:::note Cookie Persistence
For API requests (paths starting with `/v1/`) and for `/health`, locale selection via query parameter or header does **not** set a session cookie. Each API request must explicitly specify the desired locale. Cookies are only set for UI page requests, and only from the query parameter or the `X-EE-Locale` header, never from `Accept-Language`.
:::

## Contributing a translation

The catalogs live in the [`translations/`](https://github.com/postalsys/emailengine/tree/master/translations) directory of the EmailEngine repository. `messages.pot` is the template listing every translatable string, each `<locale>.po` holds one language, and the `<locale>.mo` beside it is the compiled form EmailEngine loads at runtime. A locale is served only if it is also listed in `locales.json`.

To add a language:

1. Create a new catalog in [Poedit](https://poedit.net/) from `messages.pot` (**Update from POT**)
2. Translate the strings and save the file as `<locale>.po`; Poedit compiles the `.mo` alongside it
3. Open a pull request against the repository, or send the `.po` file to andris@postalsys.com

The `README.md` in the same directory describes the equivalent GNU gettext command-line workflow, including how to refresh an existing catalog after strings change.

Validation error messages come from a separate package. Their translations are maintained in the [joi-messages](https://github.com/postalsys/joi-messages/tree/master/translations) repository.

## See Also

- [Hosted Authentication](/docs/accounts/hosted-authentication) - The public forms that translations apply to
- [Virtual Mailing Lists](/docs/advanced/virtual-lists) - The hosted unsubscribe page, another localized public page
- [API Reference Overview](/docs/api-reference/#error-handling) - The shape of the validation errors that are translated
- [Configuration Options](/docs/reference/configuration-options) - The `locale` setting among all the others

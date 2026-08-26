---
title: OpenAPI Specification
description: Download the EmailEngine OpenAPI document to generate API clients or import the API into Postman, Insomnia, and similar tools
sidebar_position: 7
---

# OpenAPI Specification

EmailEngine describes its entire HTTP API with an OpenAPI document. The same document powers the [full API reference](/docs/api/emailengine-api) on this site, so anything you can read there is also available as machine-readable JSON that API clients, code generators, and testing tools can consume directly.

Two differences. This site's reference leaves out the two deprecated [Document Store](/docs/configuration/environment-variables#advanced-settings) endpoints, `/v1/chat/{account}` and `/v1/unified/search`, which the document itself still describes until the Document Store leaves the releases on 2026-10-01. And neither describes the pre-2.79.0 token paths (`POST /v1/token`, `DELETE /v1/token/{token}`, `GET /v1/tokens/account/{account}`), which still answer for existing integrations but are deliberately left out so that new ones use `/v1/tokens`.

## Where to get the document

Every EmailEngine instance serves its own specification:

```
https://emailengine.example.com/swagger.json
```

Prefer this copy. It matches the exact version you are running, and its `servers` entry is filled in from the request, so a document downloaded from your own instance already points back at it.

A published copy of the latest release is also available if you have no instance running yet:

```
https://go.emailengine.app/swagger.json
```

This copy carries a generic placeholder as its server URL, which most tools will ask you to replace with your instance's address on import.

:::note
`/swagger.json` is intentionally unauthenticated, so the API reference page and external tooling can read it without a session. It describes the shape of the API only and exposes no account data or settings. If the endpoint should not be publicly reachable, block it at your reverse proxy.
:::

EmailEngine also renders the same document as a browsable reference in its own admin interface, under **API Reference** in the side menu (`/admin/reference`).

## What the document contains

| Property         | Value                                                                    |
| ---------------- | ------------------------------------------------------------------------ |
| OpenAPI version  | 3.0.0                                                                    |
| Paths            | Every operation of the REST API, all under `/v1`                         |
| Operation IDs    | Every operation has one, derived from method and path (`postV1Account`)  |
| Authentication   | A single `bearerAuth` scheme, applied to every operation                 |
| Tags             | One per group in the API reference sidebar                               |
| Permissions      | Each operation carries `x-ee-action` and `x-ee-group`, the grant a [narrowed token](/docs/api-reference/access-tokens#permissions) needs for it |
| Version          | The `info.version` field is the EmailEngine version that produced it     |

Server URLs in the document are origins only, so generated clients combine a base URL like `https://emailengine.example.com` with paths that already include the `/v1` prefix.

## Importing into an API client

Most API clients accept the URL directly, which keeps the import repeatable after an upgrade:

- **Postman** - Import > Link, then paste the URL. Postman creates a collection with a `baseUrl` variable and collection-level Bearer Token authentication. Set `baseUrl` to your instance and paste an [access token](/docs/api-reference/access-tokens) as the token value.
- **Insomnia** - Import from URL, then add a Bearer Token to the resulting environment.
- **Bruno** - Import Collection > OpenAPI V3, using a downloaded copy of the file.
- **Hoppscotch** - Import from URL under the collections panel.

Because authentication is declared once for the whole document, setting the token at the collection or environment level is enough. Individual requests inherit it.

## Generating a client

Any OpenAPI generator works. A few common choices:

**Typed clients in many languages** with [OpenAPI Generator](https://openapi-generator.tech/) (requires a Java runtime):

```bash
npx @openapitools/openapi-generator-cli generate \
  -i https://emailengine.example.com/swagger.json \
  -g typescript-fetch \
  -o ./emailengine-client
```

Replace `-g typescript-fetch` with `python`, `go`, `java`, `csharp`, `php`, or any other supported generator.

**TypeScript types only**, for use with your own fetch calls:

```bash
npx openapi-typescript https://emailengine.example.com/swagger.json \
  -o ./emailengine.d.ts
```

**A Python client** with [openapi-python-client](https://github.com/openapi-generators/openapi-python-client):

```bash
openapi-python-client generate \
  --url https://emailengine.example.com/swagger.json
```

Generated method names follow the operation IDs, so registering an account becomes `postV1Account` and listing messages becomes `getV1AccountAccountMessages`. Most generators let you rename these through a mapping file if the defaults read poorly in your codebase.

PHP developers can skip generation entirely and use the maintained [PHP SDK](/docs/integrations/php).

## Keeping clients current

The document changes with each EmailEngine release, mostly through added endpoints and fields. Regenerate after upgrading, and compare `info.version` in your vendored copy against the running instance to see whether anything moved:

```bash
curl -s https://emailengine.example.com/swagger.json | jq -r '.info.version'
```

Committing the generated client, rather than generating it during every build, keeps upgrades reviewable as a normal diff.

## AI coding assistants

Code generators want the full specification, but AI assistants usually work better with a condensed overview. Two purpose-built files exist for that:

- [AI Agent Reference](/docs/reference/llm-context) - endpoint list, webhook events, and common integration patterns in prose
- [capabilities.json](https://learn.emailengine.app/capabilities.json) - the same surface as structured JSON

See [AI and ChatGPT Integration](/docs/integrations/ai-chatgpt) for how these fit into an AI-assisted workflow.

## See Also

- [API Reference Overview](/docs/api-reference) - authentication, conventions, and error handling
- [Access Tokens](/docs/api-reference/access-tokens) - creating the tokens your generated client will need
- [Full API Reference](/docs/api/emailengine-api) - the same specification rendered as browsable documentation
- [Integrations Overview](/docs/integrations) - SDKs, low-code platforms, and integration patterns

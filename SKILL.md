---
name: clankerweb
description: >
  Publish static websites through a Clankerweb hosting API. Use when asked to
  publish, host, deploy, upload, share, or put a static site online with a Clankerweb
  service. Creates or updates sites through one-step /upload endpoints and returns a live URL,
  with either an API key or a temporary email-based upload flow on the same endpoint.
user-invocable: true
argument-hint: "[file-or-folder]"
allowed-tools: Bash
---

# Clankerweb

Publish static files to a Clankerweb instance and get a live URL back. Static hosting only.

## Requirements

- Required binary: `curl`
- Optional environment variables:
  - `CLANKERWEB_API_KEY`

## Authentication

Clankerweb supports two publish modes:

1. API key publishing
2. Temporary email publishing

### API key publishing

Authenticated publish requests use a Bearer token:

```bash
-H "Authorization: Bearer $CLANKERWEB_API_KEY"
```

If no API key is available, tell the user to open:

```text
https://app.clankerweb.com/agent/connect
```

They should sign up or sign in, create an upload API key, and configure it outside
chat, such as in `CLANKERWEB_API_KEY`, their usual local secret store, or another
credential mechanism supported by their agent environment.

If an API request returns `API_KEY_REQUIRED` or `MISSING_API_KEY_SCOPE`, show the `agent.connectUrl`
from the response and ask the user to configure a key with the required scopes outside chat.

Do not ask the user to paste API keys into chat. Do not print, echo, log, or return API key
values. Use authenticated publishing only when the key is already available from the
environment or an approved local secret source.

### Temporary email publishing

This is the easiest way to publish a new site without providing an API key. Use `POST /upload` and include
their email address in the `X-Email` header.

Important constraints:

- If `domain` is sent with `X-Email`, the service ignores it.
- The service generates the slug and live URL.
- The service emails a claim link to the user.
- The site stays live for 1 hour unless the user clicks the emailed link to make it permanent.

If the user asks for a custom domain or wants to update an existing site, do not use the email flow.
Use the API-key flow instead.

## File Structure

For HTML sites, publish the directory whose root contains `index.html`.

Correct:

```text
my-site/
  index.html
  styles.css
  script.js
```

Publish `my-site/`, not the parent directory containing `my-site/`.

## Publish a New Site

Use one request:

```bash
curl -sS -X POST "https://app.clankerweb.com/upload" \
  -H "Authorization: Bearer $CLANKERWEB_API_KEY" \
  -F "files=@index.html" \
  -F "files=@styles.css" \
  -F "files=@script.js" \
  -F "domain=example.clankers.page"
```

`domain` is optional. If provided, Clankerweb uses its first label as the site slug when it
matches the configured base domain. If omitted, the service generates a slug.

The response includes:

- `data.link.subdomain`
- `data.link.url`
- `data.deployment.status`
- `liveUrl`

Return the `liveUrl` to the user.

## Publish a Temporary Site By Email

Use the same `POST /upload` endpoint and send `X-Email` instead of `Authorization`:

```bash
curl -sS -X POST "https://app.clankerweb.com/upload" \
  -H "X-Email: user@example.com" \
  -F "files=@index.html" \
  -F "files=@styles.css" \
  -F "files=@script.js"
```

Optional:

- `name`
- `domain` may be present but is ignored for temporary email uploads

The response includes:

- `data.link.subdomain`
- `data.link.url`
- `data.deployment.status`
- `liveUrl`
- `temporary.email`
- `temporary.expiresAt`
- `temporary.message`

Return the `liveUrl` to the user and mention that they must click the emailed link to keep the site
permanently. Always describe the temporary site window as 1 hour.

## Update an Existing Site

Use `PUT /upload` with the existing domain:

```bash
curl -sS -X PUT "https://app.clankerweb.com/upload" \
  -H "Authorization: Bearer $CLANKERWEB_API_KEY" \
  -F "files=@index.html" \
  -F "files=@styles.css" \
  -F "files=@script.js" \
  -F "domain=example.clankers.page"
```

Updates require an existing site for that domain and a key that can create deployments.

## List Existing Sites

Before replacing an unknown site, inspect the account:

```bash
curl -sS "https://app.clankerweb.com/profile" \
  -H "Authorization: Bearer $CLANKERWEB_API_KEY"
```

The response includes `data.links`. Each link includes fields such as:

- `subdomain`
- `domain`
- `url`
- `created`
- `siteType`
- `organization`

If the user wants to replace an existing site but did not provide a domain, call `/profile`,
summarize the available sites, and ask which one to update.

## Agent Workflow

1. Resolve the folder or file the user wants published.
2. Confirm the publish root contains `index.html` for HTML sites.
3. Read `CLANKERWEB_API_KEY` if available.
4. Choose the publish mode:
5. If an API key is available from the environment or approved local secret source, use the authenticated `/upload` API.
6. If no API key is available and the user accepts a temporary generated URL, use `POST /upload` with `X-Email`.
7. If no API key is available for a custom domain or update flow, send the user to `https://app.clankerweb.com/agent/connect` and have them configure the key outside chat.
8. For a new authenticated site, call `POST /upload`.
9. For replacing a site, call `GET /profile` if needed, then `PUT /upload`.
10. Return the live URL and mention whether the deployment status is `READY`.

Use the HTTP API as the source of truth. Do not use database access or storage-bucket access to
simulate publishing.

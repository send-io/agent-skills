---
name: sendio-migrate
description: Move an application's email sending from Resend, SendGrid, Postmark, Mailgun, Mailchimp Transactional (Mandrill) or Marketing, Brevo or Mailjet to send.io with minimal code changes, by pointing the existing provider SDK or HTTP calls at send.io's compatible APIs. Use when the user wants to switch email providers to send.io, or when code uses one of those SDKs and should send through send.io.
---

# Migrate to send.io

send.io serves each provider's API shape, so most migrations are a base URL
and key swap: keep the provider SDK, change where it points and use a send.io
API key (`mx_...`). The URLs are below.

## Steps

1. Create a send.io API key; store it as `SENDIO_API_KEY` (never commit it).
2. Add and verify the sending domain (dashboard, `POST /domains`, or
   `sendio domains add example.com`), then publish the DNS records it returns.
   Sending from an unverified domain answers 403.
3. Change the base URL and key as in the table. Leave the rest of the code alone.
4. Recreate webhooks through the same compatible API (they then use that
   provider's payload format and signature), or move to native send.io webhooks.
5. Send a test to `delivered@send.io` and to an address the user owns; check
   `GET https://api.send.io/emails/{id}` or `sendio emails get <id>`.

## Base URLs and auth

| From | Base URL | Key goes in |
|---|---|---|
| Resend | `https://api.send.io` (native API, same paths) | `Authorization: Bearer mx_...` |
| SendGrid v3 | `https://api.send.io/compat/sendgrid` (then `/v3/mail/send`, ...) | `Authorization: Bearer mx_...` |
| Postmark | `https://api.send.io/compat/postmark` (then `/email`, `/email/batch`, ...) | `X-Postmark-Server-Token: mx_...` |
| Mailgun | `https://api.send.io/compat/mailgun` (then `/v3/{domain}/messages`, ...) | basic auth, user `api`, password `mx_...` |
| Mailchimp Transactional | `https://api.send.io/compat/mailchimp/api/1.0` | `"key": "mx_..."` in the JSON body |
| Mailchimp Marketing | `https://api.send.io/compat/mailchimp/3.0` | basic auth, any user, password `mx_...` |
| Brevo v3 | `https://api.send.io/compat/brevo/v3` (then `/smtp/email`, ...) | `api-key: mx_...` |
| Mailjet | `https://api.send.io/compat/mailjet` (then `/v3.1/send`, `/v3/REST/...`) | basic auth, user `mx_...`, any password |

Errors come back in the provider's own format and status codes.

## SDK changes

Resend (any Resend SDK works against the native API):

```ts
const resend = new Resend(process.env.SENDIO_API_KEY, { baseUrl: "https://api.send.io" });
// or env: RESEND_BASE_URL=https://api.send.io
```

Python `resend.api_url = "https://api.send.io"` (or `RESEND_API_URL`); Go
`client.BaseURL, _ = url.Parse("https://api.send.io/")`; Ruby
`RESEND_BASE_URL=https://api.send.io/` (trailing slash, set before require);
PHP `RESEND_BASE_URL` in the environment; Java `Resend.builder().baseUrl(...)`.

SendGrid (`@sendgrid/mail`):

```ts
sgMail.setApiKey(process.env.SENDIO_API_KEY);
sgMail.client.setDefaultRequest("baseUrl", "https://api.send.io/compat/sendgrid"); // after setApiKey
```

Postmark (`postmark`):

```ts
const client = new postmark.ServerClient(process.env.SENDIO_API_KEY, { requestHost: "api.send.io/compat/postmark" });
```

Mailgun (`mailgun.js`):

```ts
const mg = new Mailgun(formData).client({ username: "api", key: process.env.SENDIO_API_KEY, url: "https://api.send.io/compat/mailgun" });
```

Mailchimp Transactional (`@mailchimp/mailchimp_transactional`):

```ts
const mandrill = mailchimpTransactional(process.env.SENDIO_API_KEY);
mandrill.basePath = "https://api.send.io/compat/mailchimp/api/1.0";
```

Mailchimp Marketing: `mailchimp.setConfig({ apiKey: process.env.SENDIO_API_KEY, server: "server" }); mailchimp.basePath = "https://api.send.io/compat/mailchimp/3.0";`

Brevo (`@getbrevo/brevo`):

```ts
const brevo = new BrevoClient({ apiKey: process.env.SENDIO_API_KEY, baseUrl: "https://api.send.io/compat/brevo/v3" });
```

Mailjet (`node-mailjet`):

```ts
const mailjet = Mailjet.apiConnect(process.env.SENDIO_API_KEY, "unused", { config: { host: "api.send.io/compat/mailjet" } });
```

Plain HTTP: replace the provider's host with the base URL above and keep the path.

## What to check

- Each provider's reference (with per-endpoint notes on what differs):
  https://app.send.io/docs/api?guide=sendgrid (also `postmark`, `mailgun`,
  `mailchimp`, `brevo`, `mailjet`). Machine-readable specs:
  `https://app.send.io/openapi/<provider>.json`.
- Templates: create them in send.io (the provider's template endpoints work
  through the compatible API) and use the new ids.
- Suppressions and contacts do not move by themselves: import them (CSV
  import, `POST /suppressions`, or the provider's own endpoints on the compatible API).
- New code can use the native API and the `sendio` SDKs directly (see the sendio-api skill).

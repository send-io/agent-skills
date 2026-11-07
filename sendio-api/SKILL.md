---
name: sendio-api
description: Send and track email with the send.io REST API from application code. Use when writing code that sends transactional email (single or batch), sends with templates, schedules or cancels email, verifies send.io webhooks, makes retries safe with idempotency keys, or handles send.io API errors and rate limits, in Node.js, Python, Go or plain HTTP.
---

# send.io API

Base URL `https://api.send.io`. Authenticate with an API key (`mx_...`) as a
bearer token: `Authorization: Bearer $SENDIO_API_KEY`. JSON in, JSON out.

- Full reference: https://app.send.io/docs/api. Machine-readable:
  https://app.send.io/openapi.json (OpenAPI 3.1), https://app.send.io/openapi.yaml,
  Postman collection at https://app.send.io/postman/send.io.postman_collection.json.
- The API matches Resend's: the Resend SDKs work if you set their base URL to
  `https://api.send.io` (see the sendio-migrate skill).
- Keep the key in `SENDIO_API_KEY`. Never print it or commit it.

## Send an email

`POST /emails`. Required: `from` (on a verified domain, `"Name <a@acme.com>"`
works), `to` (string or array), `subject`, and `html` or `text` (or `template`).
Optional: `cc`, `bcc`, `reply_to`, `headers`, `attachments`
(`[{filename, content (base64) | path (URL), content_type}]`),
`tags` (`[{name, value}]`), `metadata` (object), `scheduled_at`
(ISO 8601 or `"in 1 hour"`, up to 30 days ahead), `topic_id`,
`track_opens`, `track_clicks`. Returns `{"id": "..."}`.

```bash
curl -X POST https://api.send.io/emails \
  -H "Authorization: Bearer $SENDIO_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: welcome-user-123" \
  -d '{"from":"Acme <hi@acme.com>","to":["ada@example.com"],"subject":"Welcome","html":"<p>Hi Ada</p>"}'
```

Node (`npm install sendio`):

```ts
import { Sendio } from "sendio";
const sendio = new Sendio(process.env.SENDIO_API_KEY);
const { data, error } = await sendio.emails.send(
  { from: "Acme <hi@acme.com>", to: ["ada@example.com"], subject: "Welcome", html: "<p>Hi Ada</p>" },
  { idempotencyKey: "welcome-user-123" },
);
if (error) throw new Error(error.message);
```

Python (`pip install sendio`):

```python
import sendio
sendio.api_key = os.environ["SENDIO_API_KEY"]  # also the default
email = sendio.Emails.send(
    {"from": "Acme <hi@acme.com>", "to": ["ada@example.com"], "subject": "Welcome", "html": "<p>Hi Ada</p>"},
    {"idempotency_key": "welcome-user-123"},
)
print(email["id"])
```

Go (`go get send.io/sendio-go`):

```go
client := sendio.NewClient(os.Getenv("SENDIO_API_KEY"))
sent, err := client.Emails.SendWithOptions(ctx, &sendio.SendEmailRequest{
	From: "Acme <hi@acme.com>", To: []string{"ada@example.com"},
	Subject: "Welcome", Html: "<p>Hi Ada</p>",
}, &sendio.SendEmailOptions{IdempotencyKey: "welcome-user-123"})
```

Test recipients: `delivered@send.io`, `bounced@send.io`, `complained@send.io`,
`suppressed@send.io` simulate outcomes (with only test recipients you may send
from `onboarding@send.io`).

## Batch

`POST /emails/batch` with a JSON array of up to 100 email objects (same fields,
no attachments). Node `sendio.batch.send([...])`, Python `sendio.Batch.send([...])`.
Header `x-batch-validation: permissive` sends the valid items and returns an
`errors` array of `{index, error}`; the default `strict` fails the whole batch.
The response has `data` (ids) and a `batch_id`
(`GET /emails/batches/{batch_id}`, `POST /emails/batches/{batch_id}/cancel`).

## Templates

1. `POST /templates` with `name`, `html`, `subject`, `from`, optional `alias`,
   `variables` (`[{key, type, fallback_value}]`), `template_engine`
   (`simple`, the default: `{{NAME}}` escaped, `{{{NAME}}}` raw; or `handlebars`:
   `#if`, `#each`, helpers).
2. `POST /templates/{id or alias}/publish` (sends use the published version).
3. Send: `POST /emails` with `"template": {"id": "order-receipt", "variables": {"NAME": "Ada"}}`
   instead of `html`/`text`. A variable with no value and no fallback fails the request.

Preview without sending: `POST /templates/{id}/render`. Versions:
`GET /templates/{id}/versions`, `POST .../versions/{version_id}/activate`.

## Track and change

- `GET /emails/{id}` (status in `last_event`), `GET /emails/{id}/events`, `GET /emails?limit=20`.
- Scheduled email: `PATCH /emails/{id}` with a new `scheduled_at`, or `POST /emails/{id}/cancel`.
- Received email (inbound): `GET /emails/receiving`, `GET /emails/receiving/{id}`.

## Webhooks

Create: `POST /webhooks` with `endpoint` (public HTTPS URL) and `events`
(e.g. `email.delivered`, `email.bounced`, `email.complained`, `email.opened`,
`email.clicked`, `email.received`). The response's `signing_secret`
(`whsec_...`) is shown once. Reply HTTP 200 or the delivery is retried.

Each delivery carries `svix-id`, `svix-timestamp` and `svix-signature`
headers. The signature is `v1,` + base64(HMAC-SHA256(key, "{svix-id}.{svix-timestamp}.{raw body}")),
where key is the base64-decoded part of the secret after `whsec_`. During a
secret rotation the header holds two space-separated signatures; accept either.
Verify against the raw body (before JSON parsing) and reject old timestamps.

```ts
// Node SDK: throws if the signature does not match
const event = sendio.webhooks.verify({
  payload: rawBody,
  headers: { id: req.headers["svix-id"], timestamp: req.headers["svix-timestamp"], signature: req.headers["svix-signature"] },
  webhookSecret: process.env.SENDIO_WEBHOOK_SECRET,
});
```

Python: `sendio.Webhooks.verify({"payload": raw, "headers": {"id": ..., "timestamp": ..., "signature": ...}, "webhook_secret": secret})`
(raises `ValueError`). Go: `client.Webhooks.Verify(&sendio.VerifyWebhookOptions{Payload: raw, Headers: sendio.WebhookHeaders{Id: ..., Timestamp: ..., Signature: ...}, WebhookSecret: secret})`.
The `svix` libraries (`new Webhook(secret).verify(body, headers)`) also work.

## Idempotency

Send `Idempotency-Key: <1 to 256 chars>` on POST (emails, batch). The same key
within 24 hours returns the first result instead of sending again. Reusing a key
with a different body returns 409 `invalid_idempotent_request`; a retry while
the first request is still running returns 409 `concurrent_idempotent_requests`.
Derive the key from your own record (`welcome-<user id>`), not a random value per attempt.

## Errors and limits

Errors are JSON: `{"statusCode": 422, "name": "validation_error", "message": "..."}`.

| Status | name | What to do |
|---|---|---|
| 400 / 422 | `validation_error`, `missing_required_field`, `invalid_attachment` | Fix the request; do not retry. |
| 401 | `missing_api_key`, `invalid_api_key`, `restricted_api_key` | Check the key; a sending-only key cannot read or manage. |
| 403 | `validation_error` (domain not verified), `invalid_access`, `missing_permission` | Verify the domain or use a key with access. |
| 404 | `not_found` | Wrong id. |
| 409 | idempotency conflicts | See above. |
| 429 | `rate_limit_exceeded`, `daily_quota_exceeded`, `monthly_quota_exceeded` | Back off and retry rate limits; quotas need a plan change. |

Rate limit: 10 requests per second per team by default; responses carry
`ratelimit-limit`, `ratelimit-remaining` and `ratelimit-reset`. Retry 429 and
5xx with exponential backoff and the same `Idempotency-Key`.

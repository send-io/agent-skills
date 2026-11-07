---
name: sendio-agent-inboxes
description: Give an AI agent its own email address with send.io agent inboxes, then read, send, reply to, forward and label email from code. Use when building an agent that receives or answers email, needs an inbox per user or task, reacts to incoming mail through webhooks, drafts replies for human approval, or signs itself up for send.io without a human-created account.
---

# send.io agent inboxes

An agent inbox is a real address (on your verified domain, or on the shared
agent domain) whose mail lands in threads you read over the API. Base URL
`https://api.send.io`, `Authorization: Bearer mx_...`.

## Create an inbox

`POST /inboxes`, all fields optional:

```bash
curl -X POST https://api.send.io/inboxes -H "Authorization: Bearer $SENDIO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"username":"support-bot","domain":"acme.com","display_name":"Acme Support","client_id":"support-bot-1"}'
```

- No `domain`: the shared agent domain (`GET /agent-domain` names it). No `username`: `agent-<random>`.
- `client_id` makes creation idempotent: the same id returns the existing inbox.
- The response has `id`, `address` and one-time IMAP/SMTP `credentials` (the
  password is an API key limited to this inbox). A taken address answers 409, an unverified domain 403.
- Group inboxes in pods (`POST /pods`, `pod_id` on create); `POST /inboxes/{id}/api-keys`
  and `POST /pods/{pod}/api-keys` mint keys scoped to one inbox or pod.

## Read

- `GET /inboxes/{id}/threads?filter=unread&label=billing&q=invoice&limit=50`
  (filters `all`, `unread`, `archived`, `trash`; page with `before=<ISO time>`).
- `GET /inboxes/{id}/threads/{thread}`: the thread with its messages.
- `GET /inboxes/{id}/messages`, `GET /inboxes/{id}/messages/{message}` (HTML, text, headers, attachments),
  `GET /inboxes/{id}/messages/{message}/attachments/{attachment}`.
- Across every inbox: `GET /inbox/threads`, `GET /inbox/threads/{id}`.

## Send and reply

- New thread: `POST /inboxes/{id}/messages` with `to` and `subject` (required), `text` and/or `html`,
  optional `cc`, `bcc`, `reply_to`, `attachments`, `labels`. The From is the inbox.
- Reply: `POST /inboxes/{id}/threads/{thread}/reply` with `text` or `html`;
  `reply_all: true` copies everyone. Threading headers are set for you.
- Forward: `POST /inboxes/{id}/threads/{thread}/forward` with `to` and an optional `text` note.
- Mark and label: `PATCH /inbox/threads/{thread}` with `read`, `archived`,
  `trashed`, `labels`, `add_labels` or `remove_labels`.

Human in the loop: `POST /inboxes/{id}/drafts` (`thread_id` to reply, or `to`
+ `subject` for a new thread; `send_at` to send on its own), list with
`GET /inboxes/{id}/drafts`, approve with `POST /inboxes/{id}/drafts/{draft}/send`.

Allow and block lists: `POST /inboxes/{id}/lists` with `direction` (`send` or
`receive`), `kind` (`allow` or `block`) and `entry` (address or domain).

## React to new mail

Webhook for one inbox: `POST /inboxes/{id}/webhooks` with `url` and
`event_types` (default `["message.received"]`; also `message.sent`,
`message.delivered`, `message.bounced`, `message.complained`,
`message.rejected`, `message.opened`, `message.received.spam`). Team-wide:
`POST /webhooks` with those `message.*` events. Deliveries are signed with
`svix-id`, `svix-timestamp`, `svix-signature` (verify as in the sendio-api skill);
answer HTTP 200.

Without a public URL, the CLI streams new messages:
`sendio inboxes watch <inbox-id-or-address> --exec ./handle.sh` (message JSON on stdin).

## Agent self sign-up

An agent with no account can create one: `POST /v0/agents/sign-up` (no key)
with optional `username` and `human_email` returns `agent_id`, `inbox_id`
(`<username>@agents.send.io`) and an `api_key`. The human gets a 6-digit code;
`POST /v0/agents/{agent_id}/verify` with `{"otp_code": "..."}` and that key
unlocks it. Until then the agent can only email its human. Limited to 20
sign-ups per hour per network.

## Other ways in

- CLI: `sendio inboxes create --username support-bot`, `sendio inboxes threads list <id> --filter unread`,
  `sendio inboxes threads reply <id> <thread> --text "..."`, `sendio inboxes messages send <id> --to a@x.dev --subject Hi --text Hello`.
- MCP tools: `create-inbox`, `list-inbox-threads`, `get-inbox-thread`,
  `send-inbox-message`, `reply-to-thread`, `forward-thread`, `update-thread`,
  `create-draft`, `send-draft` (see the sendio-mcp skill).
- AgentMail-compatible API under `/v0` (`/v0/inboxes/{inbox_id}/messages/send`, threads, drafts, pods, webhooks)
  for code written against AgentMail.

Reference: https://app.send.io/docs/api#inboxes

---
name: sendio-mcp
description: Connect an AI client (Claude Code, Claude Desktop, Cursor, Codex, VS Code and others) to send.io's hosted MCP server and use its tools to send email, check delivery, manage domains, templates, contacts, broadcasts, automations, webhooks and agent inboxes. Use when the user wants send.io available as MCP tools, or asks which send.io MCP tool to call.
---

# send.io MCP server

Hosted server (remote, Streamable HTTP): `https://api.send.io/mcp`.

Two ways to authenticate:

- **Sign in with send.io** (default): give the client only the URL. It gets a
  401 with OAuth discovery, registers itself and opens send.io in a browser,
  where the user picks the team and Full access or Sending only. Tokens last an
  hour and refresh; the app is listed (and can be revoked) on the API keys page.
- **API key**: send `Authorization: Bearer mx_...`. The key's own permission applies.

## Connect

Claude Code:

```bash
claude mcp add --transport http sendio https://api.send.io/mcp
# then /mcp in Claude Code, pick sendio, sign in. Add --scope user for every project.
# with a key instead:
claude mcp add --transport http sendio https://api.send.io/mcp --header "Authorization: Bearer $SENDIO_API_KEY"
```

Cursor (`~/.cursor/mcp.json` or `.cursor/mcp.json`):

```json
{ "mcpServers": { "sendio": { "url": "https://api.send.io/mcp" } } }
```

With a key: add `"headers": { "Authorization": "Bearer ${env:SENDIO_API_KEY}" }`.

Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.sendio]
url = "https://api.send.io/mcp"
# with a key: bearer_token_env_var = "SENDIO_API_KEY"
```

Claude Desktop / claude.ai: Customize > Connectors > Add > Add custom connector,
URL `https://api.send.io/mcp`, then Connect.

With the `sendio` CLI installed, `sendio mcp install --client <claude-code|cursor|vscode|codex|gemini|windsurf|zed|claude-desktop|claude-ai>`
writes the config for you (`--with-api-key` for a key header). `sendio mcp url`
prints the URL. `sendio mcp serve` runs the same tools locally over stdio
(`--read-only`, `--tools emails,domains`).

Setup for every client: https://app.send.io/docs/mcp

## Tools

Each tool is one send.io API call made with the connection's access, so the
API's validation and permissions apply and every call shows in Logs. Names are
kebab-case. Families:

| Family | Tools |
|---|---|
| Emails | `send-email`, `send-batch-emails`, `list-emails`, `get-email`, `update-email`, `cancel-email`, `list-email-events`, `list-email-attachments`, `get-email-attachment` |
| Received emails | `list-received-emails`, `get-received-email`, `list-received-email-attachments`, `get-received-email-attachment` |
| Templates | `create-template`, `list-templates`, `get-template`, `update-template`, `publish-template`, `duplicate-template`, `remove-template`, `render-template`, `send-test-template`, `list-template-versions`, `get-template-version`, `update-template-version`, `remove-template-version`, `activate-template-version` |
| Contacts | `create-contact`, `list-contacts`, `get-contact`, `update-contact`, `remove-contact`, `list-contact-segments`, `add-contact-to-segment`, `remove-contact-from-segment`, `list-contact-topics`, `update-contact-topics` |
| Contact imports | `create-contact-import`, `list-contact-imports`, `get-contact-import`, `start-contact-import` |
| Contact properties | `create-contact-property`, `list-contact-properties`, `get-contact-property`, `update-contact-property`, `remove-contact-property` |
| Segments | `create-segment`, `list-segments`, `get-segment`, `update-segment`, `remove-segment`, `list-segment-contacts` |
| Topics | `create-topic`, `list-topics`, `get-topic`, `update-topic`, `remove-topic` |
| Broadcasts | `create-broadcast`, `list-broadcasts`, `get-broadcast`, `update-broadcast`, `send-broadcast`, `cancel-broadcast`, `duplicate-broadcast`, `remove-broadcast`, `list-broadcast-versions`, `get-broadcast-version`, `update-broadcast-version`, `restore-broadcast-version` |
| Automations | `create-automation`, `list-automations`, `import-automation-http-step`, `get-automation`, `update-automation`, `duplicate-automation`, `stop-automation`, `remove-automation`, `list-automation-runs`, `get-automation-run`, `test-automation-http`, `get-automation-signing-secret`, `rotate-automation-signing-secret`, `list-automation-templates`, `get-automation-template`, `create-automation-from-template` |
| Events | `send-event`, `create-event`, `list-events`, `get-event`, `update-event`, `remove-event` |
| Domains | `create-domain`, `list-domains`, `get-domain`, `update-domain`, `verify-domain`, `remove-domain` |
| API keys | `create-api-key`, `list-api-keys`, `get-api-key`, `update-api-key`, `remove-api-key` |
| Webhooks | `create-webhook`, `list-webhooks`, `get-webhook`, `update-webhook`, `rotate-webhook-signing-secret`, `remove-webhook` |
| Suppressions | `add-suppression`, `list-suppressions`, `get-suppression`, `remove-suppression` |
| Logs | `list-logs`, `get-log` |
| Agent inboxes | `create-inbox`, `list-inboxes`, `get-inbox`, `delete-inbox`, `list-inbox-threads`, `get-inbox-thread`, `list-inbox-messages`, `send-inbox-message`, `reply-to-thread`, `forward-thread`, `update-thread`, `create-draft`, `list-drafts`, `send-draft` |

Tool arguments are the API's field names (`from`, `to`, `subject`, `html`,
`template`, ...); path ids are arguments of the same name. List tools take
`limit` (1 to 100), `after` and `before`.

## Working rules

- Check delivery after sending: `get-email` (its `last_event`) or `list-email-events`.
- Ask before destructive tools (`remove-*`, `delete-inbox`, `cancel-*`,
  `stop-automation`, `send-broadcast`, `rotate-webhook-signing-secret`).
- Setting up a ready-made workflow: `list-automation-templates`, then `get-automation-template` to show its emails and the params it needs (url params have no default), then `create-automation-from-template` with an `idempotencyKey`. It is created disabled: enable it with `update-automation` (`status: enabled`) only when the user asks.
- A "Sending only" connection can only send; management tools return 401/403. Tell the user instead of retrying.
- Send tests to addresses the user owns or to `delivered@send.io` / `bounced@send.io`.

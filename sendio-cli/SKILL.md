---
name: sendio-cli
description: Send and inspect email with send.io from the terminal using the `sendio` CLI. Use when the user wants to send a (test) email, check whether an email was delivered or bounced, add or verify a sending domain, manage templates, contacts, segments, broadcasts, webhooks, API keys or suppressions, read API request logs, work with agent inboxes, or call any send.io API endpoint.
---

# sendio CLI

`sendio` is send.io's official command-line client. Prefer it over hand-written
curl calls: it handles auth, pagination, retries after token refresh, dry runs
and stable exit codes.

## Agent-mode contract

When stdout is not a terminal or a coding agent is detected (CLAUDECODE,
CODEX_*, CURSOR_AGENT, GEMINI_CLI, OPENCODE, AI_AGENT), sendio runs in agent
mode (force it with `--agent yes`):

- Results are JSON on **stdout**. Lists with `--all` stream NDJSON (one item per line).
- Errors are one JSON line on **stderr**: `{"error":{"code":"…","message":"…","status":404}}`.
- It never prompts. Destructive commands (delete, cancel, stop, rotate-secret,
  `auth remove`) fail with exit 2 unless you pass `--yes`.
- `--jq '<expr>'` filters the JSON (strings print raw), e.g. `--jq '.data[].id'`.
- `--dry-run` on any write prints the exact HTTP request instead of sending it (exit 0).

Exit codes: 0 ok, 1 API error, 2 usage error, 3 auth, 4 network, 5 not found,
6 rate limited, 7 `--wait`/`--watch` timeout.

Discover everything with `sendio commands --json` (full tree with flags,
the API method/path and the MCP tool behind each command) and
`sendio <command> --help`.

## Auth

- Check first: `sendio whoami` (exit 3 means not signed in).
- `SENDIO_API_KEY=mx_… sendio …` or `--api-key mx_…` works without any saved login.
- `sendio login` opens a browser; over SSH it prints a device code. Ask the
  user to run it themselves if you cannot complete a browser sign-in.
- Profiles: `-p <profile>`, `sendio auth list|switch|status|remove|rename`.
- Never print or log API keys.
- The API is https://api.send.io; `--base-url` or `SENDIO_BASE_URL` points it
  elsewhere.

## Common tasks

Send an email and wait until it is delivered (exit 7 on timeout, 1 on bounce):

    sendio emails send --from "Acme <hi@acme.dev>" --to ada@example.com \
      --subject "Hello" --text "Hi Ada" --wait delivered --timeout 90s

- Bodies: `--html @mail.html`, `--text -` (stdin). Attachments: `--attach ./file.pdf`.
- Templates: `--template welcome --var name=Ada --var count:=3` (`:=` sends JSON-typed values).
- Later: `--scheduled-at "in 1h"`. Safe retries: `--idempotency-key auto`.
- Many at once: `sendio emails batch --file emails.ndjson`.

Inspect: `sendio emails get <id>`, `sendio emails events <id>`,
`sendio emails list --limit 20`, `sendio logs list`, `sendio logs get <id>`.

Domains:

    sendio domains add mail.example.com          # prints the DNS records
    sendio domains add mail.example.com --auto   # one-click DNS setup link when supported
    sendio domains verify <id> --watch           # until verified (exit 7 on timeout)

Templates as files (git-friendly):

    sendio templates pull ./emails
    sendio templates push ./emails --dry-run     # diff summary + planned requests
    sendio templates push ./emails --publish

Resources (all support `list --limit/--after/--all`, `get`, `create`,
`update`, `delete` where the API does): `contacts`, `contact-properties`,
`segments`, `topics`, `broadcasts` (`send --test-to a@x.dev`), `automations`
(+`runs`, `templates`: `templates use <slug> --sender from=... --params key=value`),
`events` (`send`), `webhooks` (`rotate-secret`), `api-keys`,
`suppressions`, `inboxes` (+`threads`, `messages`, `drafts`).
Flags map to API fields (`--first-name` → `first_name`); `--data @body.json`
sends a raw body and flags override its fields. Object flags take `key=value`
or JSON; list flags take repeats or commas.

CSV import: `sendio contacts import ./contacts.csv --segment <id> --wait`.

Anything else: `sendio api GET /domains --jq '.data[].name'`,
`sendio api POST /emails -f from=… -f to=… -f subject=… -f text=…`,
`sendio api /contacts --paginate`. Works for `/compat/*` paths too.

Health check: `sendio doctor` (auth, API latency, domain DNS status, webhooks).

## Rules of thumb

- Use `--json`/`--jq` and parse results; do not scrape human tables.
- Use `--dry-run` first for bulk or destructive changes and show the user the plan.
- Pass `--yes` only when the user asked for the destructive action.
- Test sends go to addresses the user owns or to `delivered@send.io` /
  `bounced@send.io` style test recipients if the account provides them.

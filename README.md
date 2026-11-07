# send.io skills for coding agents

Agent skills that teach Claude Code, Cursor, Codex and other coding agents how
to work with send.io. Each skill is a folder with a `SKILL.md` (YAML
frontmatter `name` and `description`, then instructions), the layout the
[skills CLI](https://github.com/vercel-labs/skills) installs from.

| Skill | Use it for |
|---|---|
| [`sendio-api`](sendio-api/SKILL.md) | Sending email from code: single, batch, templates, webhook verification, idempotency, errors. Node, Python, Go and HTTP. |
| [`sendio-cli`](sendio-cli/SKILL.md) | The `sendio` command-line client in agent mode. |
| [`sendio-mcp`](sendio-mcp/SKILL.md) | Connecting an AI client to the hosted MCP server, and its tool families. |
| [`sendio-migrate`](sendio-migrate/SKILL.md) | Moving from Resend, SendGrid, Postmark, Mailgun, Mailchimp, Brevo or Mailjet with a base URL swap. |
| [`sendio-agent-inboxes`](sendio-agent-inboxes/SKILL.md) | Email addresses for agents: create inboxes, read threads, reply, drafts, webhooks. |

## Install

Once this repository is published:

```bash
npx skills add sendio/skills                      # pick skills and agents interactively
npx skills add sendio/skills --skill sendio-api   # one skill
npx skills add sendio/skills --list               # list what is here
```

Locally, copy a folder into your agent's skills directory, for example
`cp -r integrations/agent-skills/sendio-api ~/.claude/skills/` (Claude Code) or
`.agents/skills/` in a project.

The `sendio` CLI installs `sendio-cli` for you: `sendio agent setup`.

## URLs

The API is `https://api.send.io`. The web app, the docs and the OpenAPI files
are on `https://app.send.io`.

## Maintenance

- `sendio-cli/SKILL.md` is a copy of `cli/internal/cli/skill/SKILL.md` (the file
  the CLI embeds for `sendio agent setup`). Edit that one and copy it here; a
  test in `cli/internal/cli` fails when they differ.
- The same test checks every skill's frontmatter (`name` matches the folder,
  lowercase with hyphens; `description` present and under 1024 characters).

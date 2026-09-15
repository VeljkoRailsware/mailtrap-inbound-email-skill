# Mailtrap skill: receiving inbound email

An [Agent Skill](https://agentskills.io) that teaches a coding agent (Claude Code, Cursor, Codex CLI, Gemini CLI, Copilot, Windsurf) how to receive email with **Mailtrap Inbound Email**: provision an inbound address, read and reply to arriving mail, verify webhook signatures, and avoid the platform quirks that are not in the docs.

Inbound Email is a distinct Mailtrap product, included with Email API/SMTP.

## Status

This skill is proposed for the official [mailtrap/mailtrap-skills](https://github.com/mailtrap/mailtrap-skills) repository as a new "Receiving" group. It is published here so it can be installed, tested and reviewed while that contribution is pending. The layout matches the official repo (`skills/<name>/SKILL.md`), so the install steps are the same.

## What it covers

- Folder → inbox → messages → threads, via the official [Mailtrap CLI](https://github.com/mailtrap/mailtrap-cli) first, with the REST API, SDKs and MCP server as named fallbacks.
- Provisioning hosted `@inbound-mailtrap.io` addresses and custom-domain catch-all inboxes.
- Reading, replying to and forwarding messages, and the guardrails around commands that send real mail.
- Webhook signature verification over the raw request body.
- Platform behaviour found only by live testing: the 10 MiB SMTP size cap, the `text_body`/`html_body` field names, expiring attachment URLs, and the fields the CLI drops on v0.6.0.

Every command and field name in the skill was written from observed output against a live inbox, not from memory or documentation.

## Install

Symlink the skill folder into the skills directory your agent reads, then restart or reload the agent.

```bash
git clone https://github.com/VeljkoRailsware/mailtrap-inbound-email-skill.git
ln -s "$(pwd)/mailtrap-inbound-email-skill/skills/receiving-inbound-email" ~/.claude/skills/
```

Other agents use the same pattern with their own directory: `~/.cursor/skills/`, `~/.agents/skills/` (Codex), `~/.gemini/skills/`, `~/.copilot/skills/`, `~/.codeium/windsurf/skills/`.

## Prerequisites

- Mailtrap CLI: `brew install mailtrap/cli/mailtrap`
- An **account-level** API token. A token scoped to one sending domain can read inbound but returns 403 when creating folders or inboxes.

## Related skills

Works alongside the official skills `sending-emails`, `testing-with-sandbox` and `authorizing-api-requests`. Its `## When not to use` section routes to them.

## License

MIT, matching the official skills repository.

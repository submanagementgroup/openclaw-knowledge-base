---
domain: security
topic: "Gateway Access Controls: DM Policy, Allowlists, Session Isolation, and Context Visibility"
type: reference
keywords:
  - dmPolicy
  - allowFrom
  - pairing
  - allowlist
  - session isolation
  - dmScope
  - per-channel-peer
  - contextVisibility
  - groupPolicy
  - DM access model
  - command authorization
source: gateway/security/index.md
related:
  - security/security-model
  - security/network-hardening
  - channels/pairing
  - channels/groups
  - gateway/configuration-reference
---

OpenClaw's access control layer has two independent dimensions: **trigger authorization** (who can send messages the bot processes) and **context visibility** (what supplemental content is injected into model input). Both must be configured to control blast radius.

## DM Access Model: Pairing, Allowlist, Open, Disabled

All DM-capable channels support a `dmPolicy` that gates inbound DMs before message processing:

- `pairing` (default): unknown senders receive a pairing code; bot ignores their message until approved. Codes expire after 1 hour; pending requests are capped at 3 per channel by default.
- `allowlist`: unknown senders are blocked (no pairing handshake).
- `open`: allow anyone to DM (public). Requires channel allowlist to include `"*"` (explicit opt-in).
- `disabled`: ignore all inbound DMs.

Approve via CLI:

```bash
openclaw pairing list <channel>
openclaw pairing approve <channel> <code>
```

## DM Session Isolation (Multi-User Mode)

By default, OpenClaw routes all DMs into the main session. If multiple people can DM the bot, isolate DM sessions:

```json5
{
  session: { dmScope: "per-channel-peer" },
}
```

This prevents cross-user context leakage while keeping group chats isolated. It is a messaging-context boundary, not a host-admin boundary. If users are mutually adversarial and share the same Gateway host/config, run separate gateways per trust boundary instead.

Session scope options:

- `session.dmScope: "main"` (default) — all DMs share one session for continuity.
- `session.dmScope: "per-channel-peer"` — each channel+sender pair gets an isolated DM context.
- `session.dmScope: "per-peer"` — each sender gets one session across all channels of the same type.
- `session.dmScope: "per-account-channel-peer"` — use when running multiple accounts on the same channel.

## Allowlists for DMs and Groups

OpenClaw has two separate "who can trigger me?" layers:

**DM allowlist** (`allowFrom`, `channels.discord.allowFrom`, `channels.slack.allowFrom`):
- Controls who is allowed to talk to the bot in direct messages.
- When `dmPolicy="pairing"`, approvals are written to `~/.openclaw/credentials/<channel>-allowFrom.json` (default account) or `<channel>-<accountId>-allowFrom.json` (non-default accounts), merged with config allowlists.

**Group allowlist** (channel-specific):
- `channels.whatsapp.groups`, `channels.telegram.groups`, `channels.imessage.groups`: per-group defaults including `requireMention`. When set, also acts as a group allowlist (include `"*"` to keep allow-all behavior).
- `groupPolicy="allowlist"` + `groupAllowFrom`: restrict who can trigger the bot inside a group session.
- `channels.discord.guilds` / `channels.slack.channels`: per-surface allowlists + mention defaults.
- Group checks run in order: `groupPolicy`/group allowlists first, mention/reply activation second.
- Replying to a bot message does **not** bypass sender allowlists like `groupAllowFrom`.

Treat `dmPolicy="open"` and `groupPolicy="open"` as last-resort settings. Prefer pairing + allowlists unless you fully trust every member of the room.

## Context Visibility Model

Two separate concepts:

- **Trigger authorization**: who can trigger the agent (`dmPolicy`, `groupPolicy`, allowlists, mention gates).
- **Context visibility**: what supplemental context is injected into model input (reply body, quoted text, thread history, forwarded metadata).

`contextVisibility` settings:
- `contextVisibility: "all"` (default) — keeps supplemental context as received.
- `contextVisibility: "allowlist"` — filters supplemental context to senders allowed by the active allowlist checks.
- `contextVisibility: "allowlist_quote"` — like `allowlist`, but still keeps one explicit quoted reply.

Set `contextVisibility` per channel or per room/conversation. Claims that only show "model can see quoted text from non-allowlisted senders" are hardening findings addressable with `contextVisibility`, not auth bypasses.

## Command Authorization Model

Slash commands and directives are only honored for **authorized senders**. Authorization is derived from channel allowlists/pairing plus `commands.useAccessGroups`. If a channel allowlist is empty or includes `"*"`, commands are effectively open for that channel.

`/exec` is a session-only convenience for authorized operators. It does **not** write config or change other sessions.

## Control Plane Tools Risk

Two built-in tools can make persistent control-plane changes:

- `gateway` can inspect config with `config.schema.lookup`/`config.get`, and make persistent changes with `config.apply`, `config.patch`, and `update.run`.
- `cron` can create scheduled jobs that keep running after the original chat/task ends.

The owner-only `gateway` runtime tool refuses to rewrite `tools.exec.ask` or `tools.exec.security`. Agent-driven `gateway config.apply` and `gateway config.patch` edits are fail-closed by default.

For any agent/surface that handles untrusted content, deny these by default:

```json5
{
  tools: {
    deny: ["gateway", "cron", "sessions_spawn", "sessions_send"],
  },
}
```

## Shared Inbox Quick Rule

If more than one person can DM your bot:

- Set `session.dmScope: "per-channel-peer"` (or `"per-account-channel-peer"` for multi-account channels).
- Keep `dmPolicy: "pairing"` or strict allowlists.
- Never combine shared DMs with broad tool access.

## Credential Storage Map

Use when auditing access or deciding what to back up:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram bot token**: config/env or `channels.telegram.tokenFile` (regular file only; symlinks rejected)
- **Discord bot token**: config/env or SecretRef
- **Slack tokens**: config/env (`channels.slack.*`)
- **Pairing allowlists**: `~/.openclaw/credentials/<channel>-allowFrom.json` (default account)
- **Model auth profiles**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **Codex runtime state**: `~/.openclaw/agents/<agentId>/agent/codex-home/`
- **File-backed secrets payload (optional)**: `~/.openclaw/secrets.json`
- **Legacy OAuth import**: `~/.openclaw/credentials/oauth.json`

## Node Execution (system.run)

If a macOS node is paired, the Gateway can invoke `system.run` on that node — this is remote code execution on the Mac:

- Requires node pairing (approval + token).
- Gateway node pairing establishes node identity/trust and token issuance; it is not a per-command approval surface.
- The Gateway applies a coarse global node command policy via `gateway.nodes.allowCommands` / `denyCommands`.
- Controlled on the Mac via Settings → Exec approvals (`security` + `ask` + allowlist).
- If you don't want remote execution, set security to **deny** and remove node pairing.

## Dynamic Skills (Watcher / Remote Nodes)

OpenClaw can refresh the skills list mid-session:

- **Skills watcher**: changes to `SKILL.md` can update the skills snapshot on the next agent turn.
- **Remote nodes**: connecting a macOS node can make macOS-only skills eligible.

Treat skill folders as **trusted code** and restrict who can modify them.

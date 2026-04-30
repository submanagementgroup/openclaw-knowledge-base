---
domain: cli
topic: "CLI: Nodes, Devices, Pairing, QR Code, and Message Commands"
type: reference
keywords:
  - openclaw node
  - openclaw nodes
  - openclaw devices
  - openclaw pairing
  - openclaw message
  - pairing CLI
related:
  - nodes/nodes-overview
  - gateway/gateway-features
  - cli/cli-overview
source:
  - cli/node.md
  - cli/devices.md
  - cli/pairing.md
  - cli/qr.md
  - cli/message.md
---

CLI commands for nodes, devices, pairing, and messaging.

## Node CLI (openclaw node)

# `openclaw node`

Run a **headless node host** that connects to the Gateway WebSocket and exposes
`system.run` / `system.which` on this machine.

## Why use a node host?

Use a node host when you want agents to **run commands on other machines** in your
network without installing a full macOS companion app there.

Common use cases:

- Run commands on remote Linux/Windows boxes (build servers, lab machines, NAS).
- Keep exec **sandboxed** on the gateway, but delegate approved runs to other hosts.
- Provide a lightweight, headless execution target for automation or CI nodes.

Execution is still guarded by **exec approvals** and per‑agent allowlists on the
node host, so you can keep command access scoped and explicit.

## Browser proxy (zero-config)

Node hosts automatically advertise a browser proxy if `browser.enabled` is not
disabled on the node. This lets the agent use browser automation on that node
without extra configuration.

By default, the proxy exposes the node's normal browser profile surface. If you
set `nodeHost.browserProxy.allowProfiles`, the proxy becomes restrictive:
non-allowlisted profile targeting is rejected, and persistent profile
create/delete routes are blocked through the proxy.

Disable it on the node if needed:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## Run (foreground)

```bash
openclaw node run --host <gateway-host> --port 18789
```

Options:

- `--host <host>`: Gateway WebSocket host (default: `127.0.0.1`)
- `--port <port>`: Gateway WebSocket port (default: `18789`)
- `--tls`: Use TLS for the gateway connection
- `--tls-fingerprint <sha256>`: Expected TLS certificate fingerprint (sha256)
- `--node-id <id>`: Override node id (clears pairing token)
- `--display-name <name>`: Override the node display name

## Gateway auth for node host

`openclaw node run` and `openclaw node install` resolve gateway auth from config/env (no `--token`/`--password` flags on node commands):

- `OPENCLAW_GATEWAY_T

## Nodes CLI (openclaw nodes)

# `openclaw nodes`

Manage paired nodes (devices) and invoke node capabilities.

Related:

- Nodes overview: [Nodes](/nodes)
- Camera: [Camera nodes](/nodes/camera)
- Images: [Image nodes](/nodes/images)

Common options:

- `--url`, `--token`, `--timeout`, `--json`

## Common commands

```bash
openclaw nodes list
openclaw nodes list --connected
openclaw nodes list --last-connected 24h
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
```

`nodes list` prints pending/paired tables. Paired rows include the most recent connect age (Last Connect).
Use `--connected` to only show currently-connected nodes. Use `--last-connected <duration>` to
filter to nodes that connected within a duration (e.g. `24h`, `7d`).
Use `nodes remove --node <id|name|ip>` to delete a stale gateway-owned node pairing record.

Approval note:

- `openclaw nodes pending` only needs pairing scope.
- `gateway.nodes.pairing.autoApproveCidrs` can skip the pending step only for
  explicitly trusted, first-time `role: node` device pairing. It is off by
  default and does not approve upgrades.
- `openclaw nodes approve <requestId>` inherits extra scope requirements from the
  pending request:
  - commandless request: pairing only
  - non-exec node commands: pairing + write
  - `system.run` / `system.run.prepare` / `system.which`: pairing + admin

## Invoke

```bash
openclaw nodes invoke --node <id|name|ip> --command <command> --params <json>
```

Invoke flags:

- `--params <json>`: JSON object string (default `{}`).
- `--invoke-timeout <ms>`: node invoke timeout (default `15000`).
- `--idempotency-key <key>`: optional idempotency key.
- `system.run` and `system.run.prepare` are blocked here; use the `exec` tool with `host=node` for shell execution.

For shell execution on a node, use the `exec` tool with `host=node` instead of `openclaw nodes run`.
The `nodes` CLI is now capability-focused: direct RPC via `nodes invoke`, plus pairing, camera,
screen, location, canvas, and notifications.

## Related

- [CLI reference](/cli)
- [Nodes](/nodes)

## Devices CLI (openclaw devices)

# `openclaw devices`

Manage device pairing requests and device-scoped tokens.

## Commands

### `openclaw devices list`

List pending pairing requests and paired devices.

```
openclaw devices list
openclaw devices list --json
```

Pending request output shows the requested access next to the device's current
approved access when the device is already paired. This makes scope/role
upgrades explicit instead of looking like the pairing was lost.

### `openclaw devices remove <deviceId>`

Remove one paired device entry.

When you are authenticated with a paired device token, non-admin callers can
remove only **their own** device entry. Removing some other device requires
`operator.admin`.

```
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

### `openclaw devices clear --yes [--pending]`

Clear paired devices in bulk.

```
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

### `openclaw devices approve [requestId] [--latest]`

Approve a pending device pairing request by exact `requestId`. If `requestId`
is omitted or `--latest` is passed, OpenClaw only prints the selected pending
request and exits; rerun approval with the exact request ID after verifying
the details.

If a device retries pairing with changed auth details (role, scopes, or public key), OpenClaw supersedes the previous pending entry and issues a new `requestId`. Run `openclaw devices list` right before approval to use the current ID.

If the device is already paired and asks for broader scopes or a broader role,
OpenClaw keeps the existing approval in place and creates a new pending upgrade
request. Review the `Requested` vs `Approved` columns in `openclaw devices list`
or use `openclaw devices approve --latest` to preview the exact upgrade before
approving it.

If the Gateway is explicitly configured with
`gateway.nodes.pairing.autoApproveCidrs`, first-time `role: node` requests from
matching client IPs

## Pairing CLI (openclaw pairing)

# `openclaw pairing`

Approve or inspect DM pairing requests (for channels that support pairing).

Related:

- Pairing flow: [Pairing](/channels/pairing)

## Commands

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

List pending pairing requests for one channel.

Options:

- `[channel]`: positional channel id
- `--channel <channel>`: explicit channel id
- `--account <accountId>`: account id for multi-account channels
- `--json`: machine-readable output

Notes:

- If multiple pairing-capable channels are configured, you must provide a channel either positionally or with `--channel`.
- Extension channels are allowed as long as the channel id is valid.

## `pairing approve`

Approve a pending pairing code and allow that sender.

Usage:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>` when exactly one pairing-capable channel is configured

Options:

- `--channel <channel>`: explicit channel id
- `--account <accountId>`: account id for multi-account channels
- `--notify`: send a confirmation back to the requester on the same channel

Owner bootstrap:

- If `commands.ownerAllowFrom` is empty when you approve a pairing code, OpenClaw also records the approved sender as the command owner, using a channel-scoped entry such as `telegram:123456789`.
- This only bootstraps the first owner. Later pairing approvals do not replace or expand `commands.ownerAllowFrom`.
- The command owner is the human operator account allowed to run owner-only commands and approve dangerous actions such as `/diagnostics`, `/export-trajectory`, `/config`, and exec approvals.

## Notes

- Channel input: pass it positionally (`pairing list telegram`) or with `--channel <channel>`.
- `pairing list` supports `--account <accountId>` for multi-account channels.
- `pairing approve` supports `--account <accountId>` and `--notify`.
- If only one pairing-capable channel is configured, `pairing approve <code>` is allowed.
- If you approved a sender before this bootstrap existed, run `openclaw doctor`; it warns when no command owner is configured and shows the `openclaw config set commands.ownerAllowFrom ...` command to fix it.

## Related

- [CLI reference](/cli)
- [Channel pairing](/channels/pairing)

## QR Code CLI (openclaw qr)

# `openclaw qr`

Generate a mobile pairing QR and setup code from your current Gateway configuration.

## Usage

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --url wss://gateway.example/ws
```

## Options

- `--remote`: prefer `gateway.remote.url`; if it is unset, `gateway.tailscale.mode=serve|funnel` can still provide the remote public URL
- `--url <url>`: override gateway URL used in payload
- `--public-url <url>`: override public URL used in payload
- `--token <token>`: override which gateway token the bootstrap flow authenticates against
- `--password <password>`: override which gateway password the bootstrap flow authenticates against
- `--setup-code-only`: print only setup code
- `--no-ascii`: skip ASCII QR rendering
- `--json`: emit JSON (`setupCode`, `gatewayUrl`, `auth`, `urlSource`)

## Notes

- `--token` and `--password` are mutually exclusive.
- The setup code itself now carries an opaque short-lived `bootstrapToken`, not the shared gateway token/password.
- In the built-in node/operator bootstrap flow, the primary node token still lands with `scopes: []`.
- If bootstrap handoff also issues an operator token, it stays bounded to the bootstrap allowlist: `operator.approvals`, `operator.read`, `operator.talk.secrets`, `operator.write`.
- Bootstrap scope checks are role-prefixed. That operator allowlist only satisfies operator requests; non-operator roles still need scopes under their own role prefix.
- Mobile pairing fails closed for Tailscale/public `ws://` gateway URLs. Private LAN `ws://` remains supported, but Tailscale/public mobile routes should use Tailscale Serve/Funnel or a `wss://` gateway URL.
- With `--remote`, OpenClaw requires either `gateway.remote.url` or
  `gateway.tailscale.mode=serve|funnel`.
- With `--remote`, if effectively active remote credentials are configured as SecretRefs and you do not pass `--token` or `--password`, the command resolves them from the active gateway snapshot. If gateway is unavailable, the command fails fast.
- Without `--remote`, local gateway auth SecretRefs are resolved when no CLI auth override is passed:
  - `gateway.auth.token` resolves when token auth can win (explicit `gateway.auth.mode="token"` or inferred mode where no password source wins).
  - `gateway.auth.password` resolves when password auth can win (explicit `gateway.auth.mode="password"` or inferred mode with no winning token from auth/env).
- If both `gateway.auth.token` and `gateway.auth.password` are configured (including SecretRefs) and `gateway.auth.mode` is unset, setup-code resolution fails until mode is set explicitly.
- Gateway version skew note: this command path requires a gateway that supports `secrets.resolve`; older gateways return an unknown-method error.
- After scanning, approve device pairing with:
  - `openclaw devices list`
  - `openclaw devices approve <requestId>`

## Related

- [CLI reference](/cli)
- [Pairing](/cli/pairing)

## Message CLI (openclaw message)

# `openclaw message`

Single outbound command for sending messages and channel actions
(Discord/Google Chat/iMessage/Matrix/Mattermost (plugin)/Microsoft Teams/Signal/Slack/Telegram/WhatsApp).

## Usage

```
openclaw message <subcommand> [flags]
```

Channel selection:

- `--channel` required if more than one channel is configured.
- If exactly one channel is configured, it becomes the default.
- Values: `discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp` (Mattermost requires plugin)
- `openclaw message` resolves the selected channel to its owning plugin when `--channel` or a channel-prefixed target is present; otherwise it loads configured channel plugins for default-channel inference.

Target formats (`--target`):

- WhatsApp: E.164 or group JID
- Telegram: chat id or `@username`
- Discord: `channel:<id>` or `user:<id>` (or `<@id>` mention; raw numeric ids are treated as channels)
- Google Chat: `spaces/<spaceId>` or `users/<userId>`
- Slack: `channel:<id>` or `user:<id>` (raw channel id is accepted)
- Mattermost (plugin): `channel:<id>`, `user:<id>`, or `@username` (bare ids are treated as channels)
- Signal: `+E.164`, `group:<id>`, `signal:+E.164`, `signal:group:<id>`, or `username:<name>`/`u:<name>`
- iMessage: handle, `chat_id:<id>`, `chat_guid:<guid>`, or `chat_identifier:<id>`
- Matrix: `@user:server`, `!room:server`, or `#alias:server`
- Microsoft Teams: conversation id (`19:...@thread.tacv2`) or `conversation:<id>` or `user:<aad-object-id>`

Name lookup:

- For supported providers (Discord/Slack/etc), channel names like `Help` or `#help` are resolved via the directory cache.
- On cache miss, OpenClaw will attempt a live directory lookup when the provider supports it.

## Common flags

- `--channel <name>`
- `--account <id>`
- `--target <dest>` (target channel or user for send/poll/read/etc)
- `--targets <name>` (repeat; broadcast only)
- `--json`
- `--dry-run`
- `--verbose`

## SecretRef behavior

- `openclaw message` resolves supported channel SecretRefs before running the selected action.
- Resolution is scoped to the active action target when possible:
  - channel-scoped when `--channel` is set (or inferred from prefixed targets like `discord:...`)
  - account-scoped when `--account` is set (channel globals + selected account surfaces)
  - when `--account` is omitted, OpenClaw does not force a `default` account SecretRef scope
- Unresolved SecretRefs on unrelated channels do not block a targeted message action.
- If the selected channel/account SecretRef is unresolved, the command fails closed for that action.

## Actions

### Core

- `send`
  - Channels: WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (plugin)/Signal/iMessage/Matrix/Microsoft Teams
  - Required: `--target`, plus `--message`, `--media`, or `--presentation`
  - Optional: `--media`, `--presentation`, `--delivery`, `--pin`, `--reply-to`, `--thread-id`, `--gif-playback`, `--force-document`, `--silent`
  - Shared presentation payl

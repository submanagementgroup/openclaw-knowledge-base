---
domain: cli
topic: "CLI Commands: nodes, pairing, proxy..."
type: reference
keywords:
  - nodes
  - pairing
  - proxy
  - qr
  - reset
  - CLI commands
  - openclaw CLI
source: 
  - cli/nodes.md
  - cli/pairing.md
  - cli/proxy.md
  - cli/qr.md
  - cli/reset.md
  - cli/sandbox.md
  - cli/secrets.md
  - cli/security.md
---

## openclaw nodes Command

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
screen, location, Canvas, and notifications. Canvas commands are implemented by the bundled experimental Canvas plugin; core keeps a compatibility hook so they remain under `openclaw nodes canvas`.

## Related

- [CLI reference](/cli)
- [Nodes](/nodes)


## openclaw pairing Command

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


## openclaw proxy Command

# `openclaw proxy`

Validate operator-managed proxy routing, or run the local explicit debug proxy
and inspect captured traffic.

Use `validate` to preflight an operator-managed forward proxy before enabling
OpenClaw proxy routing. The other commands are debugging tools for
transport-level investigation: they can start a local proxy, run a child command
with capture enabled, list capture sessions, query common traffic patterns, read
captured blobs, and purge local capture data.

## Commands

```bash
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy validate [--json] [--proxy-url <url>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

## Validate

`openclaw proxy validate` checks the effective operator-managed proxy URL from
`--proxy-url`, config, or `OPENCLAW_PROXY_URL`. It reports a config problem when
no proxy is enabled and configured; use `--proxy-url` for a one-off preflight
before changing config. By default it verifies that a public destination succeeds
through the proxy and that the proxy cannot reach a temporary loopback canary.
Custom denied destinations are fail-closed: HTTP responses and ambiguous
transport failures both fail unless you can verify a deployment-specific denial
signal separately. Add `--apns-reachable` to also open an APNs HTTP/2 CONNECT
tunnel through the proxy and confirm sandbox APNs responds; the probe uses an
intentionally invalid provider token, so an APNs `403 InvalidProviderToken`
response is a successful reachability signal.

Options:

- `--json`: print machine-readable JSON.
- `--proxy-url <url>`: validate this proxy URL instead of config or env.
- `--allowed-url <url>`: add a destination expected to succeed through the proxy. Repeat to check multiple destinations.
- `--denied-url <url>`: add a destination expected to be blocked by the proxy. Repeat to check multiple destinations.
- `--apns-reachable`: also verify sandbox APNs HTTP/2 is reachable through the proxy.
- `--apns-authority <url>`: APNs authority to probe with `--apns-reachable` (`https://api.sandbox.push.apple.com` by default; production is `https://api.push.apple.com`).
- `--timeout-ms <ms>`: per-request timeout in milliseconds.

See [Network Proxy](/security/network-proxy) for deployment guidance and denial
semantics.

## Query presets

`openclaw proxy query --preset <name>` accepts:

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

## Notes

- `start` defaults to `127.0.0.1` unless `--host` is set.
- `run` starts a local debug proxy and then runs the command after `--`.
- The debug proxy's direct upstream forwarding opens upstream sockets for diagnostics. When OpenClaw managed proxy mode is active, direct forwarding for proxy requests and CONNECT tunnels is disabled by default; set `OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1` only for approved local diagnostics.
- `validate` exits with code 1 when proxy config or destination checks fail.
- Captures are local debugging data; use `openclaw proxy purge` when finished.

## Related

- [CLI reference](/cli)
- [Network Proxy](/security/network-proxy)
- [Trusted proxy auth](/gateway/trusted-proxy-auth)


## openclaw qr Command (QR Code)

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
- Mobile pairing fails closed for Tailscale/public `ws://` gateway URLs. Private LAN addresses and `.local` Bonjour hosts remain supported over `ws://`, but Tailscale/public mobile routes should use Tailscale Serve/Funnel or a `wss://` gateway URL.
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


## openclaw reset Command

# `openclaw reset`

Reset local config/state (keeps the CLI installed).

Options:

- `--scope <scope>`: `config`, `config+creds+sessions`, or `full`
- `--yes`: skip confirmation prompts
- `--non-interactive`: disable prompts; requires `--scope` and `--yes`
- `--dry-run`: print actions without removing files

Examples:

```bash
openclaw backup create
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

Notes:

- Run `openclaw backup create` first if you want a restorable snapshot before removing local state.
- If you omit `--scope`, `openclaw reset` uses an interactive prompt to choose what to remove.
- `--non-interactive` is only valid when both `--scope` and `--yes` are set.

## Related

- [CLI reference](/cli)


## openclaw sandbox Command

Manage sandbox runtimes for isolated agent execution.

## Overview

OpenClaw can run agents in isolated sandbox runtimes for security. The `sandbox` commands help you inspect and recreate those runtimes after updates or configuration changes.

Today that usually means:

- Docker sandbox containers
- SSH sandbox runtimes when `agents.defaults.sandbox.backend = "ssh"`
- OpenShell sandbox runtimes when `agents.defaults.sandbox.backend = "openshell"`

For `ssh` and OpenShell `remote`, recreate matters more than with Docker:

- the remote workspace is canonical after the initial seed
- `openclaw sandbox recreate` deletes that canonical remote workspace for the selected scope
- next use seeds it again from the current local workspace

## Commands

### `openclaw sandbox explain`

Inspect the **effective** sandbox mode/scope/workspace access, sandbox tool policy, and elevated gates (with fix-it config key paths).

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

### `openclaw sandbox list`

List all sandbox runtimes with their status and configuration.

```bash
openclaw sandbox list
openclaw sandbox list --browser  # List only browser containers
openclaw sandbox list --json     # JSON output
```

**Output includes:**

- Runtime name and status
- Backend (`docker`, `openshell`, etc.)
- Config label and whether it matches current config
- Age (time since creation)
- Idle time (time since last use)
- Associated session/agent

### `openclaw sandbox recreate`

Remove sandbox runtimes to force recreation with updated config.

```bash
openclaw sandbox recreate --all                # Recreate all containers
openclaw sandbox recreate --session main       # Specific session
openclaw sandbox recreate --agent mybot        # Specific agent
openclaw sandbox recreate --browser            # Only browser containers
openclaw sandbox recreate --all --force        # Skip confirmation
```

**Options:**

- `--all`: Recreate all sandbox containers
- `--session <key>`: Recreate container for specific session
- `--agent <id>`: Recreate containers for specific agent
- `--browser`: Only recreate browser containers
- `--force`: Skip confirmation prompt

> **Note:** Runtimes are automatically recreated when the agent is next used.


## Use cases

### After updating a Docker image

```bash
# Pull new image
docker pull openclaw-sandbox:latest
docker tag openclaw-sandbox:latest openclaw-sandbox:bookworm-slim

# Update config to use new image
# Edit config: agents.defaults.sandbox.docker.image (or agents.list[].sandbox.docker.image)

# Recreate containers
openclaw sandbox recreate --all
```

### After changing sandbox configuration

```bash
# Edit config: agents.defaults.sandbox.* (or agents.list[].sandbox.*)

# Recreate to apply new config
openclaw sandbox recreate --all
```

### After changing SSH target or SSH auth material

```bash
# Edit config:
# - agents.defaults.sandbox.backend
# - agents.defaults.sandbox.ssh.target
# - agents.defaults.sandbox.ssh.workspaceRoot
# - agents.defaults.sandbox.ssh.identityFile / certificateFile / knownHostsFile
# - agents.defaults.sandbox.ssh.identityData / certificateData / knownHostsData

openclaw sandbox recreate --all
```

For the core `ssh` backend, recreate deletes the per-scope remote workspace root
on the SSH target. The next run seeds it again from the local workspace.

### After changing OpenShell source, policy, or mode

```bash
# Edit config:
# - agents.defaults.sandbox.backend
# - plugins.entries.openshell.config.from
# - plugins.entries.openshell.config.mode
# - plugins.entries.openshell.config.policy

openclaw sandbox recreate --all
```

For OpenShell `remote` mode, recreate deletes the canonical remote workspace
for that scope. The next run seeds it again from the local workspace.

### After changing setupCommand

```bash
openclaw sandbox recreate --all
# or just one agent:
openclaw sandbox recreate --agent family
```

### For a specific agent only

```bash
# Update only one agent's containers
openclaw sandbox recreate --agent alfred
```

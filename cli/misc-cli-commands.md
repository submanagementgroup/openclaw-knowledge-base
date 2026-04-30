---
domain: cli
topic: "Miscellaneous CLI Commands: flows, tui, voicecall, doctor, dns, webhooks, and More"
type: reference
keywords:
  - openclaw flows
  - openclaw tui
  - openclaw doctor
  - openclaw dns
  - openclaw webhooks
  - CLI miscellaneous
related:
  - cli/cli-overview
  - gateway/health-diagnostics-logging
source:
  - cli/flows.md
  - cli/tui.md
  - cli/doctor.md
  - cli/dns.md
  - cli/webhooks.md
---

Additional CLI commands: flows, tui, voicecall, doctor, DNS, webhooks, and more.

## Flows (openclaw flows)
# `openclaw tasks flow`

Flow commands are subcommands of `openclaw tasks`, not a standalone `flows` command.

```bash
openclaw tasks flow list [--json]
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

For full documentation see [Task Flow](/automation/taskflow) and the [tasks CLI reference](/cli/tasks).

## Related

- [CLI reference](/cli)
- [Automation](/automation)

## Clawbot (openclaw clawbot)
# `openclaw clawbot`

Legacy alias namespace kept for backwards compatibility.

Current supported alias:

- `openclaw clawbot qr` (same behavior as [`openclaw qr`](/cli/qr))

## Migration

Prefer modern top-level commands directly:

- `openclaw clawbot qr` -> `openclaw qr`

## Related

- [CLI reference](/cli)

## Docs (openclaw docs)
# `openclaw docs`

Search the live docs index.

Arguments:

- `[query...]`: search terms to send to the live docs index

Examples:

```bash
openclaw docs
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

Notes:

- With no query, `openclaw docs` opens the live docs search entrypoint.
- Multi-word queries are passed through as one search request.

## Related

- [CLI reference](/cli)

## DNS (openclaw dns)
# `openclaw dns`

DNS helpers for wide-area discovery (Tailscale + CoreDNS). Currently focused on macOS + Homebrew CoreDNS.

Related:

- Gateway discovery: [Discovery](/gateway/discovery)
- Wide-area discovery config: [Configuration](/gateway/configuration)

## Setup

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

## `dns setup`

Plan or apply CoreDNS setup for unicast DNS-SD discovery.

Options:

- `--domain <domain>`: wide-area discovery domain (for example `openclaw.internal`)
- `--apply`: install or update CoreDNS config and restart the service (requires sudo; macOS only)

What it shows:

- resolved discovery domain
- zone file path
- current tailnet IPs
- recommended `openclaw.json` discovery config
- the Tailscale Split DNS nameserver/domain values to set

Notes:

- Without `--apply`, the command is a planning helper only and prints the recommended setup.
- If `--domain` is omitted, OpenClaw uses `discovery.wideArea.domain` from config.
- `--apply` currently supports macOS only and expects Homebrew CoreDNS.
- `--apply` bootstraps the zone file if needed, ensures the CoreDNS import stanza exists, and restarts the `coredns` brew service.

## Related

- [CLI reference](/cli)
- [Discovery](/gateway/discovery)

## Proxy (openclaw proxy)
# `openclaw proxy`

Run the local explicit debug proxy and inspect captured traffic.

This is a debugging command for transport-level investigation. It can start a
local proxy, run a child command with capture enabled, list capture sessions,
query common traffic patterns, read captured blobs, and purge local capture
data.

## Commands

```bash
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

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
- Captures are local debugging data; use `openclaw proxy purge` when finished.

## Related

- [CLI reference](/cli)
- [Trusted proxy auth](/gateway/trusted-proxy-auth)

## Voice Call (openclaw voicecall)
# `openclaw voicecall`

`voicecall` is a plugin-provided command. It only appears if the voice-call plugin is installed and enabled.

Primary doc:

- Voice-call plugin: [Voice Call](/plugins/voice-call)

## Common commands

```bash
openclaw voicecall setup
openclaw voicecall smoke
openclaw voicecall status --call-id <id>
openclaw voicecall call --to "+15555550123" --message "Hello" --mode notify
openclaw voicecall continue --call-id <id> --message "Any questions?"
openclaw voicecall dtmf --call-id <id> --digits "ww123456#"
openclaw voicecall end --call-id <id>
```

`setup` prints human-readable readiness checks by default. Use `--json` for
scripts:

```bash
openclaw voicecall setup --json
```

For external providers (`twilio`, `telnyx`, `plivo`), setup must resolve a public
webhook URL from `publicUrl`, a tunnel, or Tailscale exposure. A loopback/private
serve fallback is rejected because carriers cannot reach it.

`smoke` runs the same readiness checks. It will not place a real phone call
unless both `--to` and `--yes` are present:

```bash
openclaw voicecall smoke --to "+15555550123"        # dry run
openclaw voicecall smoke --to "+15555550123" --yes  # live notify call
```

## Exposing webhooks (Tailscale)

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

Security note: only expose the webhook endpoint to networks you trust. Prefer Tailscale Serve over Funnel when possible.

## Related

- [CLI reference](/cli)
- [Voice call plugin](/plugins/voice-call)

## TUI (openclaw tui)
# `openclaw tui`

Open the terminal UI connected to the Gateway, or run it in local embedded
mode.

Related:

- TUI guide: [TUI](/web/tui)

Notes:

- `chat` and `terminal` are aliases for `openclaw tui --local`.
- `--local` cannot be combined with `--url`, `--token`, or `--password`.
- `tui` resolves configured gateway auth SecretRefs for token/password auth when possible (`env`/`file`/`exec` providers).
- When launched from inside a configured agent workspace directory, TUI auto-selects that agent for the session key default (unless `--session` is explicitly `agent:<id>:...`).
- Local mode uses the embedded agent runtime directly. Most local tools work, but Gateway-only features are unavailable.
- Local mode adds `/auth [provider]` inside the TUI command surface.
- Plugin approval gates still apply in local mode. Tools that require approval prompt for a decision in the terminal; nothing is silently auto-approved because the Gateway is not involved.

## Examples

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "Compare my config to the docs and tell me what to fix"
# when run inside an agent workspace, infers that agent automatically
openclaw tui --session bugfix
```

## Config repair loop

Use local mode when the current config already validates and you want the
embedded agent to inspect it, compare it against the docs, and help repair it
from the same terminal:

If `openclaw config validate` is already failing, use `openclaw configure` or
`openclaw doctor --fix` first. `openclaw chat` does not bypass the invalid-
config guard.

```bash
openclaw chat
```

Then inside the TUI:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

Apply targeted fixes with `openclaw config set` or `openclaw configure`, then
rerun `openclaw config validate`. See [TUI](/web/tui) and [Config](/c

## Doctor (openclaw doctor)
# `openclaw doctor`

Health checks + quick fixes for the gateway and channels.

Related:

- Troubleshooting: [Troubleshooting](/gateway/troubleshooting)
- Security audit: [Security](/gateway/security)

## Examples

```bash
openclaw doctor
openclaw doctor --repair
openclaw doctor --deep
openclaw doctor --repair --non-interactive
openclaw doctor --generate-gateway-token
```

## Options

- `--no-workspace-suggestions`: disable workspace memory/search suggestions
- `--yes`: accept defaults without prompting
- `--repair`: apply recommended repairs without prompting
- `--fix`: alias for `--repair`
- `--force`: apply aggressive repairs, including overwriting custom service config when needed
- `--non-interactive`: run without prompts; safe migrations only
- `--generate-gateway-token`: generate and configure a gateway token
- `--deep`: scan system services for extra gateway installs

Notes:

- Interactive prompts (like keychain/OAuth fixes) only run when stdin is a TTY and `--non-interactive` is **not** set. Headless runs (cron, Telegram, no terminal) will skip prompts.
- Performance: non-interactive `doctor` runs skip eager plugin loading so headless health checks stay fast. Interactive sessions still fully load plugins when a check needs their contribution.
- `--fix` (alias for `--repair`) writes a backup to `~/.openclaw/openclaw.json.bak` and drops unknown config keys, listing each removal.
- State integrity checks now detect orphan transcript files in the sessions directory. Archiving them as `.deleted.<timestamp>` requires an interactive confirmation; `--fix`, `--yes`, and headless runs leave them in place.
- Doctor also scans `~/.openclaw/cron/jobs.json` (or `cron.store`) for legacy cron job shapes and can rewrite them in place before the scheduler has to auto-normalize them at runtime.
- Doctor repairs missing bundled plugin runtime dependencies without writing into packaged global installs. For root-owned npm installs or hardened systemd units, set `OPENCLAW_PLUGIN_

## System (openclaw system)
# `openclaw system`

System-level helpers for the Gateway: enqueue system events, control heartbeats,
and view presence.

All `system` subcommands use Gateway RPC and accept the shared client flags:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--expect-final`

## Common commands

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
openclaw system event --text "Check for urgent follow-ups" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

Enqueue a system event on the **main** session. The next heartbeat will inject
it as a `System:` line in the prompt. Use `--mode now` to trigger the heartbeat
immediately; `next-heartbeat` waits for the next scheduled tick.

Flags:

- `--text <text>`: required system event text.
- `--mode <mode>`: `now` or `next-heartbeat` (default).
- `--json`: machine-readable output.
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags.

## `system heartbeat last|enable|disable`

Heartbeat controls:

- `last`: show the last heartbeat event.
- `enable`: turn heartbeats back on (use this if they were disabled).
- `disable`: pause heartbeats.

Flags:

- `--json`: machine-readable output.
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags.

## `system presence`

List the current system presence entries the Gateway knows about (nodes,
instances, and similar status lines).

Flags:

- `--json`: machine-readable output.
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags.

## Notes

- Requires a running Gateway reachable by your current config (local or remote).
- System events are ephemeral and not persisted across restarts.

## Related

- [CLI reference](/cli)

## Webhooks (openclaw webhooks)
# `openclaw webhooks`

Webhook helpers and integrations (Gmail Pub/Sub, webhook helpers).

Related:

- Webhooks: [Webhooks](/automation/cron-jobs#webhooks)
- Gmail Pub/Sub: [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration)

## Gmail

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail run
```

### `webhooks gmail setup`

Configure Gmail watch, Pub/Sub, and OpenClaw webhook delivery.

Required:

- `--account <email>`

Options:

- `--project <id>`
- `--topic <name>`
- `--subscription <name>`
- `--label <label>`
- `--hook-url <url>`
- `--hook-token <token>`
- `--push-token <token>`
- `--bind <host>`
- `--port <port>`
- `--path <path>`
- `--include-body`
- `--max-bytes <n>`
- `--renew-minutes <n>`
- `--tailscale <funnel|serve|off>`
- `--tailscale-path <path>`
- `--tailscale-target <target>`
- `--push-endpoint <url>`
- `--json`

Examples:

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

### `webhooks gmail run`

Run `gog watch serve` plus the watch auto-renew loop.

Options:

- `--account <email>`
- `--topic <topic>`
- `--subscription <name>`
- `--label <label>`
- `--hook-url <url>`
- `--hook-token <token>`
- `--push-token <token>`
- `--bind <host>`
- `--port <port>`
- `--path <path>`
- `--include-body`
- `--max-bytes <n>`
- `--renew-minutes <n>`
- `--tailscale <funnel|serve|off>`
- `--tailscale-path <path>`
- `--tailscale-target <target>`

Example:

```bash
openclaw webhooks gmail run --account you@example.com
```

See [Gmail Pub/Sub documentation](/automation/cron-jobs#gmail-pubsub-integration) for the end-to-end setup flow and operational details.

## Related

- [CLI reference](/cli)
- [Webhook automation](/automation/webhook)

## Directory (openclaw directory)
# `openclaw directory`

Directory lookups for channels that support it (contacts/peers, groups, and “me”).

## Common flags

- `--channel <name>`: channel id/alias (required when multiple channels are configured; auto when only one is configured)
- `--account <id>`: account id (default: channel default)
- `--json`: output JSON

## Notes

- `directory` is meant to help you find IDs you can paste into other commands (especially `openclaw message send --target ...`).
- For many channels, results are config-backed (allowlists / configured groups) rather than a live provider directory.
- Default output is `id` (and sometimes `name`) separated by a tab; use `--json` for scripting.

## Using results with `message send`

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## ID formats (by channel)

- WhatsApp: `+15551234567` (DM), `1234567890-1234567890@g.us` (group)
- Telegram: `@username` or numeric chat id; groups are numeric ids
- Slack: `user:U…` and `channel:C…`
- Discord: `user:<id>` and `channel:<id>`
- Matrix (plugin): `user:@user:server`, `room:!roomId:server`, or `#alias:server`
- Microsoft Teams (plugin): `user:<id>` and `conversation:<id>`
- Zalo (plugin): user id (Bot API)
- Zalo Personal / `zalouser` (plugin): thread id (DM/group) from `zca` (`me`, `friend list`, `group list`)

## Self ("me")

```bash
openclaw directory self --channel zalouser
```

## Peers (contacts/users)

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## Groups

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```

## Related

- [CLI reference](/cli)

## Commitments (openclaw commitments)
List and manage inferred follow-up commitments.

Commitments are opt-in, short-lived follow-up memories created from
conversation context. See [Inferred commitments](/concepts/commitments) for the
conceptual guide.

With no subcommand, `openclaw commitments` lists pending commitments.

## Usage

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## Options

- `--all`: show all statuses instead of only pending commitments.
- `--agent <id>`: filter to one agent id.
- `--status <status>`: filter by status. Values: `pending`, `sent`,
  `dismissed`, `snoozed`, or `expired`.
- `--json`: output machine-readable JSON.

## Examples

List pending commitments:

```bash
openclaw commitments
```

List every stored commitment:

```bash
openclaw commitments --all
```

Filter to one agent:

```bash
openclaw commitments --agent main
```

Find snoozed commitments:

```bash
openclaw commitments --status snoozed
```

Dismiss one or more commitments:

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

Export as JSON:

```bash
openclaw commitments --all --json
```

## Output

Text output includes:

- commitment id
- status
- kind
- earliest due time
- scope
- suggested check-in text

JSON output also includes the commitment store path and full stored records.

## Related

- [Inferred commitments](/concepts/commitments)
- [Memory overview](/concepts/memory)
- [Heartbeat](/gateway/heartbeat)
- [Scheduled tasks](/automation/cron-jobs)

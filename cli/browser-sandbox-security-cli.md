---
domain: cli
topic: "CLI: Browser, Sandbox, Approvals, and Security Commands"
type: reference
keywords:
  - openclaw browser
  - openclaw sandbox
  - openclaw approvals
  - openclaw security
  - browser CLI
  - sandbox CLI
related:
  - gateway/sandboxing
  - tools/exec-approvals
  - cli/cli-overview
source:
  - cli/browser.md
  - cli/sandbox.md
  - cli/approvals.md
  - cli/security.md
---

CLI commands for browser, sandbox, approvals, and security.

## Browser CLI (openclaw browser)

# `openclaw browser`

Manage OpenClaw's browser control surface and run browser actions (lifecycle, profiles, tabs, snapshots, screenshots, navigation, input, state emulation, and debugging).

Related:

- Browser tool + API: [Browser tool](/tools/browser)

## Common flags

- `--url <gatewayWsUrl>`: Gateway WebSocket URL (defaults to config).
- `--token <token>`: Gateway token (if required).
- `--timeout <ms>`: request timeout (ms).
- `--expect-final`: wait for a final Gateway response.
- `--browser-profile <name>`: choose a browser profile (default from config).
- `--json`: machine-readable output (where supported).

## Quick start (local)

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

Agents can run the same readiness check with `browser({ action: "doctor" })`.

## Quick troubleshooting

If `start` fails with `not reachable after start`, troubleshoot CDP readiness first. If `start` and `tabs` succeed but `open` or `navigate` fails, the browser control plane is healthy and the failure is usually navigation SSRF policy.

Minimal sequence:

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

Detailed guidance: [Browser troubleshooting](/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## Lifecycle

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

Notes:

- `doctor --deep` adds a live snapshot probe. It is useful when basic CDP
  readiness is green but you want proof that the current tab can be inspected.
- For `attachOnly` and remote CDP profiles, `openclaw browser stop` closes the
  active control session and clears temporary emulation overrides even when
  OpenClaw did not launch the browser process itself.
- For local managed profiles, `openclaw browser stop` stops the spawned browser
  process.
- `openclaw browser start --headless` applies only to that start request and
  only when OpenClaw launches a local managed browser. It does not rewrite
  `browser.headless` or profile config, and it is a no-op for an already-running
  browser.
- On Linux hosts without `DISPLAY` or `WAYLAND_DISPLAY`, local managed profiles
  run headless automatically unless `OPENCLAW_BROWSER_HEADLESS=0`,
  `browser.headless=false`, or `browser.profiles.<name>.headless=false`
  explicitly requests a visible browser.

## If the command is missing

If `openclaw browser` is an unknown command, check `plugins.allow` in
`~/.openclaw/openclaw.json`.

When `plugins.allow` is present, list the bundled browser plugin explicitly
unless the config already 

## Sandbox CLI (openclaw sandbox)

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

Runtimes are automatically recreated when the agent is next used.

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
# - a

## Approvals CLI (openclaw approvals)

# `openclaw approvals`

Manage exec approvals for the **local host**, **gateway host**, or a **node host**.
By default, commands target the local approvals file on disk. Use `--gateway` to target the gateway, or `--node` to target a specific node.

Alias: `openclaw exec-approvals`

Related:

- Exec approvals: [Exec approvals](/tools/exec-approvals)
- Nodes: [Nodes](/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` is the local convenience command for keeping the requested
`tools.exec.*` config and the local host approvals file aligned in one step.

Use it when you want to:

- inspect the local requested policy, host approvals file, and effective merge
- apply a local preset such as YOLO or deny-all
- synchronize local `tools.exec.*` and local `~/.openclaw/exec-approvals.json`

Examples:

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

Output modes:

- no `--json`: prints the human-readable table view
- `--json`: prints machine-readable structured output

Current scope:

- `exec-policy` is **local-only**
- it updates the local config file and the local approvals file together
- it does **not** push policy to the gateway host or a node host
- `--host node` is rejected in this command because node exec approvals are fetched from the node at runtime and must be managed through node-targeted approvals commands instead
- `openclaw exec-policy show` marks `host=node` scopes as node-managed at runtime instead of deriving an effective policy from the local approvals file

If you need to edit remote host approvals directly, keep using `openclaw approvals set --gateway`
or `openclaw approvals set --node <id|name|ip>`.

## Common commands

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
```

`openclaw approvals get` now shows the effective exec policy for local, gateway, and node targets:

- requested `tools.exec` policy
- host approvals-file policy
- effective result after precedence rules are applied

Precedence is intentional:

- the host approvals file is the enforceable source of truth
- requested `tools.exec` policy can narrow or broaden intent, but the effective result is still derived from the host rules
- `--node` combines the node host approvals file with gateway `tools.exec` policy, because both still apply at runtime
- if gateway config is unavailable, the CLI falls back to the node approvals snapshot and notes that the final runtime policy could not be computed

## Replace approvals from a file

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`s

## Security CLI (openclaw security)

# `openclaw security`

Security tools (audit + optional fixes).

Related:

- Security guide: [Security](/gateway/security)

## Audit

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --fix
openclaw security audit --json
```

The audit warns when multiple DM senders share the main session and recommends **secure DM mode**: `session.dmScope="per-channel-peer"` (or `per-account-channel-peer` for multi-account channels) for shared inboxes.
This is for cooperative/shared inbox hardening. A single Gateway shared by mutually untrusted/adversarial operators is not a recommended setup; split trust boundaries with separate gateways (or separate OS users/hosts).
It also emits `security.trust_model.multi_user_heuristic` when config suggests likely shared-user ingress (for example open DM/group policy, configured group targets, or wildcard sender rules), and reminds you that OpenClaw is a personal-assistant trust model by default.
For intentional shared-user setups, the audit guidance is to sandbox all sessions, keep filesystem access workspace-scoped, and keep personal/private identities or credentials off that runtime.
It also warns when small models (`<=300B`) are used without sandboxing and with web/browser tools enabled.
For webhook ingress, it warns when `hooks.token` reuses the Gateway token, when `hooks.token` is short, when `hooks.path="/"`, when `hooks.defaultSessionKey` is unset, when `hooks.allowedAgentIds` is unrestricted, when request `sessionKey` overrides are enabled, and when overrides are enabled without `hooks.allowedSessionKeyPrefixes`.
It also warns when sandbox Docker settings are configured while sandbox mode is off, when `gateway.nodes.denyCommands` uses ineffective pattern-like/unknown entries (exact node command-name matching only, not shell-text filtering), when `gateway.nodes.allowCommands` explicitly enables dangerous node commands, when global `tools.profile="minimal"` is overridden by agent tool profiles, when open groups expose runtime/filesystem tools without sandbox/workspace guards, and when installed plugin tools may be reachable under permissive tool policy.
It also flags `gateway.allowRealIpFallback=true` (header-spoofing risk if proxies are misconfigured) and `discovery.mdns.mode="full"` (metadata leakage via mDNS TXT records).
It also warns when sandbox browser uses Docker `bridge` network without `sandbox.browser.cdpSourceRange`.
It also flags dangerous sandbox Docker network modes (including `host` and `container:*` namespace joins).
It also warns when existing sandbox browser Docker containers have missing/stale hash labels (for example pre-migration containers missing `openclaw.browserConfigEpoch`) and recommends `openclaw sandbox recreate --browser --all`.
It also warns when npm-based plugin/hook install records are unpinned, missing integrity metadata, or drift from currently in

---
domain: gateway
topic: "Gateway Features: Background Process, Device Pairing, OpenShell, Gateway Lock"
type: reference
keywords:
  - background process
  - device pairing
  - OpenShell
  - gateway lock
  - CLI backends
  - pairing QR
  - paired devices
related:
  - gateway/gateway-runbook
  - nodes/nodes-overview
source:
  - gateway/background-process.md
  - gateway/pairing.md
  - gateway/openshell.md
  - gateway/gateway-lock.md
---

Additional gateway features: background process management, device pairing, OpenShell backend, and gateway locking.

## Background Process

# Background Exec + Process Tool

OpenClaw runs shell commands through the `exec` tool and keeps long‑running tasks in memory. The `process` tool manages those background sessions.

## exec tool

Key parameters:

- `command` (required)
- `yieldMs` (default 10000): auto‑background after this delay
- `background` (bool): background immediately
- `timeout` (seconds, default `tools.exec.timeoutSec`): kill the process after this timeout; set `timeout: 0` only to disable the exec process timeout for that call
- `elevated` (bool): run outside the sandbox if elevated mode is enabled/allowed (`gateway` by default, or `node` when the exec target is `node`)
- Need a real TTY? Set `pty: true`.
- `workdir`, `env`

Behavior:

- Foreground runs return output directly.
- When backgrounded (explicit or timeout), the tool returns `status: "running"` + `sessionId` and a short tail.
- Background and `yieldMs` runs inherit `tools.exec.timeoutSec` unless the call provides an explicit `timeout`.
- Output is kept in memory until the session is polled or cleared.
- If the `process` tool is disallowed, `exec` runs synchronously and ignores `yieldMs`/`background`.
- Spawned exec commands receive `OPENCLAW_SHELL=exec` for context-aware shell/profile rules.
- For long-running work that starts now, start it once and rely on automatic
  completion wake when it is enabled and the command emits output or fails.
- If automatic completion wake is unavailable, or you need quiet-success
  confirmation for a command that exited cleanly without output, use `process`
  to confirm completion.
- Do not emulate reminders or delayed follow-ups with `sleep` loops or repeated
  polling; use cron for future work.

## Child process bridging

When spawning long-running child processes outside the exec/process tools (for example, CLI respawns or gateway helpers), attach the child-process bridge helper so termination signals are forwarded and listeners are detached on exit/error. This avoids orphaned processes on sy

## Gateway Pairing

Device pairing allows nodes (iOS, Android) and remote CLIs to connect to the Gateway securely:

```bash
openclaw pairing create    # generate a pairing QR code
openclaw pairing list      # list paired devices
openclaw pairing revoke <id>  # revoke a device
```

In Gateway-owned pairing, the **Gateway** is the source of truth for which nodes
are allowed to join. UIs (macOS app, future clients) are just frontends that
approve or reject pending requests.

**Important:** WS nodes use **device pairing** (role `node`) during `connect`.
`node.pair.*` is a separate pairing store and does **not** gate the WS handshake.
Only clients that explicitly call `node.pair.*` use this flow.

## Concepts

- **Pending request**: a node asked to join; requires approval.
- **Paired node**: approved node with an issued auth token.
- **Transport**: the Gateway WS endpoint forwards requests but does not decide
  membership. (Legacy TCP bridge support has been removed.)

## How pairing works

1. A node connects to the Gateway WS and requests pairing.
2. The Gateway stores a **pending request** and emits `node.pair.requested`.
3. You approve or reject the request (CLI or UI).
4. On approval, the Gateway issues a **new token** (tokens are rotated on re‑pair).
5. The node reconnects using the token and is now “paired”.

Pending requests expire automatically after **5 minutes**.

## CLI workflow (headless friendly)

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status` shows paired/connected nodes and their capabilities.

## API surface (gateway protocol)

Events:

- `node.pair.requested` — emitted when a new pending request is created.
- `node.pair.resolved` — emitted when a request is approved/rejected/expired.

Methods:

- `node.pair.request` — create or reuse a pending request.
- `node.pair.list` — list pending + paired nodes (`operator.pairing`).
- `node.pair.approve` — approve a pending request (issues token).
- `node.pair.reject` — reject a pending request.
- `node.pair.remove` — remove a stale paired node entry.
- `node.pair.verify` — verify `{ nodeId

## OpenShell Backend

OpenShell is an alternative CLI backend for running OpenClaw commands:

OpenShell is a managed sandbox backend for OpenClaw. Instead of running Docker
containers locally, OpenClaw delegates sandbox lifecycle to the `openshell` CLI,
which provisions remote environments with SSH-based command execution.

The OpenShell plugin reuses the same core SSH transport and remote filesystem
bridge as the generic [SSH backend](/gateway/sandboxing#ssh-backend). It adds
OpenShell-specific lifecycle (`sandbox create/get/delete`, `sandbox ssh-config`)
and an optional `mirror` workspace mode.

## Prerequisites

- The `openshell` CLI installed and on `PATH` (or set a custom path via
  `plugins.entries.openshell.config.command`)
- An OpenShell account with sandbox access
- OpenClaw Gateway running on the host

## Quick start

1. Enable the plugin and set the sandbox backend:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

2. Restart the Gateway. On the next agent turn, OpenClaw creates an OpenShell
   sandbox and routes tool execution through it.

3. Verify:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## Workspace modes

This is the most important decision when using OpenShell.

### `mirror`

Use `plugins.entries.openshell.config.mode: "mirror"` when you want the **local
workspace to stay canonical**.

Behavior:

- Before `exec`, OpenClaw syncs the local workspace into the OpenShell sandbox.
- After `exec`, OpenClaw syncs the remote workspace back to the local workspace.
- File tools still operate through the sandbox bridge, but the local workspace
  remains the source of truth between turns.

Best for:

- You edit files locally outside OpenClaw and want those changes visible in the
  sandbox automatically.
- You want the Open

## Gateway Lock

The gateway lock prevents concurrent gateway starts on the same port:

## Why

- Ensure only one gateway instance runs per base port on the same host; additional gateways must use isolated profiles and unique ports.
- Survive crashes/SIGKILL without leaving stale lock files.
- Fail fast with a clear error when the control port is already occupied.

## Mechanism

- The gateway first acquires a per-config lock file under the state lock directory and probes the configured port for an existing listener.
- If the recorded lock owner is gone, the port is free, or the lock is stale, startup reclaims the lock and continues.
- The gateway then binds the HTTP/WebSocket listener (default `ws://127.0.0.1:18789`) using an exclusive TCP listener.
- If the bind fails with `EADDRINUSE`, startup throws `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- On shutdown the gateway closes the HTTP/WebSocket server and removes the lock file.

## Error surface

- If another process holds the port, startup throws `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- Other bind failures surface as `GatewayLockError("failed to bind gateway socket on ws://127.0.0.1:<port>: …")`.

## Operational notes

- If the port is occupied by _another_ process, the error is the same; free the port or choose another with `openclaw gateway --port <port>`.
- Under a service supervisor, a new gateway process that sees an existing healthy `/healthz` responder exits successfully and leaves that process in control. If the existing process never becomes healthy, retries are bounded and startup fails with a clear lock error instead of looping forever.
- The macOS app still maintains its own lightweight PID guard before spawning the gateway; the runtime lock is enforced by the lock file plus HTTP/WebSocket bind.

## Related

- [Multiple Gateways](/gateway/multiple-gateways) — running multiple instances with unique ports
- [Troubleshooting](/gateway/troubleshooting) — diagnosing `EADDRINUSE` and port conflicts

## CLI Backends

OpenClaw can run **local AI CLIs** as a **text-only fallback** when API providers are down,
rate-limited, or temporarily misbehaving. This is intentionally conservative:

- **OpenClaw tools are not injected directly**, but backends with `bundleMcp: true`
  can receive gateway tools via a loopback MCP bridge.
- **JSONL streaming** for CLIs that support it.
- **Sessions are supported** (so follow-up turns stay coherent).
- **Images can be passed through** if the CLI accepts image paths.

This is designed as a **safety net** rather than a primary path. Use it when you
want “always works” text responses without relying on external APIs.

If you want a full harness runtime with ACP session controls, background tasks,
thread/conversation binding, and persistent external coding sessions, use
[ACP Agents](/tools/acp-agents) instead. CLI backends are not ACP.

## Beginner-friendly quick start

You can use Codex CLI **without any config** (the bundled OpenAI plugin
registers a default backend):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.5
```

If your gateway runs under launchd/systemd and PATH is minimal, add just the
command path:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

That’s it. No keys, no extra auth config needed beyond the CLI itself.

If you use a bundled CLI backend as the **primary message provider** on a
gateway host, OpenClaw now auto-loads the owning bundled plugin when your config
explicitly references that backend in a model ref or under
`agents.defaults.cliBackends`.

## Using it as a fallback

Add a CLI backend to your fallback list so it only runs when primary models fail:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.5"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/

---
domain: gateway
topic: "Gateway Operations: Startup, Runtime Model, OpenAI-Compatible Endpoints, Supervision, and Operational Checks"
type: procedure
keywords:
  - gateway startup
  - openclaw gateway
  - launchd
  - systemd
  - hot reload
  - OpenAI compatible
  - v1/chat/completions
  - v1/models
  - gateway status
  - supervisor
  - gateway install
  - gateway restart
  - gateway port
  - bind mode
  - loopback
  - gateway runbook
source: gateway/index.md
related:
  - gateway/configuration-overview
  - gateway/health
  - gateway/troubleshooting
  - gateway/multiple-gateways
  - gateway/remote-access
  - gateway/secrets
  - gateway/background-process
---

Gateway operations runbook for day-1 startup and day-2 management. The Gateway is a single always-on process for routing, control plane, and channel connections.

## 5-Minute Local Startup

```bash
# Start the Gateway
openclaw gateway --port 18789

# With debug/trace mirrored to stdio
openclaw gateway --port 18789 --verbose

# Force-kill listener on selected port, then start
openclaw gateway --force
```

### Verify Service Health

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Healthy baseline: `Runtime: running`, `Connectivity probe: ok`, and `Capability:` matching what you expect. Use `openclaw gateway status --require-rpc` when you need read-scope RPC proof, not just reachability.

### Validate Channel Readiness

```bash
openclaw channels status --probe
```

With a reachable gateway this runs live per-account channel probes and optional audits. If the gateway is unreachable, the CLI falls back to config-only channel summaries.

## Runtime Model

- One always-on process for routing, control plane, and channel connections.
- Single multiplexed port for: WebSocket control/RPC, HTTP APIs including OpenAI-compatible surface, Control UI, and hooks.
- Default bind mode: `loopback`.
- Auth is required by default. Use `gateway.auth.token` / `gateway.auth.password` (or `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`) for shared-secret setups.

### Port and Bind Precedence

| Setting | Resolution order |
|---|---|
| Gateway port | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789` |
| Bind mode | CLI/override → `gateway.bind` → `loopback` |

After changing `gateway.port`, run `openclaw doctor --fix` or `openclaw gateway install --force` so launchd/systemd/schtasks starts the process on the new port.

## OpenAI-Compatible Endpoints

The highest-leverage compatibility surface:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Why this set matters:
- Most Open WebUI, LobeChat, and LibreChat integrations probe `/v1/models` first.
- Many RAG and memory pipelines expect `/v1/embeddings`.
- Agent-native clients increasingly prefer `/v1/responses`.

Planning notes:
- `/v1/models` is agent-first: returns `openclaw`, `openclaw/default`, and `openclaw/<agentId>`.
- `openclaw/default` is the stable alias that always maps to the configured default agent.
- Use `x-openclaw-model` when you want a backend provider/model override; otherwise the selected agent's normal model and embedding setup stays in control.

All endpoints run on the main Gateway port and use the same trusted operator auth boundary as the rest of the Gateway HTTP API.

## Hot Reload Modes

Config reload watches the active config file path. Default mode is `gateway.reload.mode="hybrid"`.

| `gateway.reload.mode` | Behavior |
|---|---|
| `off` | No config reload |
| `hot` | Apply only hot-safe changes |
| `restart` | Restart on reload-required changes |
| `hybrid` (default) | Hot-apply when safe, restart when required |

After the first successful load, the running process serves the active in-memory config snapshot; successful reload swaps that snapshot atomically.

## Operator Command Set

```bash
openclaw gateway status
openclaw gateway status --deep   # adds a system-level service scan
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` is for extra service discovery (LaunchDaemons/systemd system units/schtasks), not a deeper RPC health probe.

## Supervision and Service Lifecycle

Use supervised runs for production-like reliability.

### macOS (launchd)

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

Use `openclaw gateway restart` for restarts. Do not chain `openclaw gateway stop` and `openclaw gateway start` as a restart substitute.

On macOS, `gateway stop` uses `launchctl bootout` by default — this removes the LaunchAgent from the current boot session without persisting a disable, so KeepAlive auto-recovery still works after unexpected crashes. To persistently suppress auto-respawn across reboots, pass `--disable`: `openclaw gateway stop --disable`.

LaunchAgent labels are `ai.openclaw.gateway` (default) or `ai.openclaw.<profile>` (named profile). `openclaw doctor` audits and repairs service config drift.

### Linux (systemd user)

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

For persistence after logout, enable lingering:

```bash
sudo loginctl enable-linger <user>
```

Manual user-unit example:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group

[Install]
WantedBy=default.target
```

### Linux (system service, multi-user/always-on hosts)

Install under `/etc/systemd/system/openclaw-gateway[-<profile>].service`. Do not also let `openclaw doctor --fix` install a user-level gateway service for the same profile/port — use `OPENCLAW_SERVICE_REPAIR_POLICY=external` when the system unit owns the lifecycle.

### Windows (native)

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Uses a Scheduled Task named `OpenClaw Gateway` (or `OpenClaw Gateway (<profile>)` for named profiles). Falls back to a per-user Startup-folder launcher if Scheduled Task creation is denied.

## Dev Profile Quick Path

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Defaults include isolated state/config and base gateway port `19001`.

## Protocol Quick Reference (Operator View)

- First client frame must be `connect`.
- Gateway returns `hello-ok` snapshot (`presence`, `health`, `stateVersion`, `uptimeMs`, limits/policy).
- `hello-ok.features.methods` / `events` are a conservative discovery list.
- Requests: `req(method, params)` → `res(ok/payload|error)`.
- Common events: `connect.challenge`, `agent`, `chat`, `session.message`, `session.tool`, `sessions.changed`, `presence`, `tick`, `health`, `heartbeat`, pairing/approval lifecycle events, `shutdown`.

Agent runs are two-stage:
1. Immediate accepted ack (`status:"accepted"`)
2. Final completion response (`status:"ok"|"error"`), with streamed `agent` events in between.

## Operational Checks

### Liveness

Open WS and send `connect`. Expect `hello-ok` response with snapshot.

### Readiness

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### Gap Recovery

Events are not replayed. On sequence gaps, refresh state (`health`, `system-presence`) before continuing.

## Common Failure Signatures

| Signature | Likely issue |
|---|---|
| `refusing to bind gateway ... without auth` | Non-loopback bind without a valid gateway auth path |
| `another gateway instance is already listening` / `EADDRINUSE` | Port conflict |
| `Gateway start blocked: set gateway.mode=local` | Config set to remote mode, or local-mode stamp is missing |
| `unauthorized` during connect | Auth mismatch between client and gateway |

For full diagnosis ladders, use [Gateway Troubleshooting](/gateway/troubleshooting).

## Safety Guarantees

- Gateway protocol clients fail fast when Gateway is unavailable (no implicit direct-channel fallback).
- Invalid/non-connect first frames are rejected and closed.
- Graceful shutdown emits `shutdown` event before socket close.

## Remote Access Quick Reference

Preferred: Tailscale/VPN.
Fallback: SSH tunnel.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

SSH tunnels do not bypass gateway auth. Clients must still send `token`/`password` even over the tunnel. See [Remote Gateway](/gateway/remote-access) and [Tailscale](/gateway/tailscale).

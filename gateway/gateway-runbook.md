---
domain: gateway
topic: "Gateway Service Runbook: Start, Status, Daemon, and Channel Probes"
type: procedure
keywords:
  - gateway
  - openclaw gateway
  - daemon
  - service
  - startup
  - openclaw status
  - channel probes
related:
  - gateway/configuration-overview
  - gateway/troubleshooting
  - gateway/health
source: gateway/index.md
---

The OpenClaw Gateway is the central process that manages all channel connections, agent sessions, and routing. Start it with `openclaw gateway` and manage it with `openclaw gateway status`, `openclaw logs`, and `openclaw channels status`.

## Starting the Gateway

```bash
openclaw gateway --port 18789
openclaw gateway --port 18789 --verbose   # debug output to stdio
openclaw gateway --force                   # kill existing listener then start
```

## Checking Gateway Health

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw gateway status --require-rpc     # confirm RPC is live, not just reachable
```

Healthy baseline: `Runtime: running`, `Connectivity probe: ok`, `Capability: ...` matching your config.

## Channel Status

```bash
openclaw channels status --probe    # live per-account channel probes
```

## Daemon (Background Service)

```bash
openclaw daemon install     # install as launchd/systemd service
openclaw daemon start
openclaw daemon stop
openclaw daemon status
openclaw daemon uninstall
```

## Gateway Locking

Only one Gateway runs per port. If another process holds the port, use `--force` or stop the old one via `openclaw daemon stop`.

Use this page for day-1 startup and day-2 operations of the Gateway service.

    Symptom-first diagnostics with exact command ladders and log signatures.

    Task-oriented setup guide + full configuration reference.

    SecretRef contract, runtime snapshot behavior, and migrate/reload operations.

    Exact `secrets apply` target/path rules and ref-only auth-profile behavior.

## 5-minute local startup

```bash
openclaw gateway --port 18789
# debug/trace mirrored to stdio
openclaw gateway --port 18789 --verbose
# force-kill listener on selected port, then start
openclaw gateway --force
```

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Healthy baseline: `Runtime: running`, `Connectivity probe: ok`, and `Capability: ...` that matches what you expect. Use `openclaw gateway status --require-rpc` when you need read-scope RPC proof, not just reachability.

```bash
openclaw channels status --probe
```

With a reachable gateway this runs live per-account channel probes and optional audits.
If the gateway is unreachable, the CLI falls back to config-only channel summaries instead
of live probe output.

Gateway config reload watches the active config file path (resolved from profile/state defaults, or `OPENCLAW_CONFIG_PATH` when set).
Default mode is `gateway.reload.mode="hybrid"`.
After the first successful load, the running process serves the active in-memory config snapshot; successful reload swaps that snapshot atomically.

## Runtime model

- One always-on process for routing, control plane, and channel connections.
- Single multiplexed port for:
  - WebSocket control/RPC
  - HTTP APIs, OpenAI compatible (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - Control UI and hooks
- Default bind mode: `loopback`.
- Auth is required by default. Shared-secret setups use
  `gateway.auth.token` / `gateway.auth.password` (or
  `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`), and non-loopback
  reverse-proxy setups can use `gateway.auth.mode: "trusted-proxy"`.

## OpenAI-compatible endpoints

OpenClaw’s highest-leverage compatibility surface is now:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Why this set matters:

- Most Open WebUI, LobeChat, and LibreChat integrations probe `/v1/models` first.
- Many RAG and memory pipelines expect `/v1/embeddings`.
- Agent-native clients increasingly prefer `/v1/responses`.

Planning note:

- `/v1/models` is agent-first: it returns `openclaw`, `openclaw/default`, and `openclaw/<agentId>`.
- `openclaw/default` is the stable alias that always maps to the configured default agent.
- Use `x-openclaw-model` when you want a backend provider/model override; otherwise the selected agent's normal model and embedding setup stays in control.

All of these run on the main Gateway port and use the same trusted operator auth boundary as the rest of the Gateway HTTP API.

### Port

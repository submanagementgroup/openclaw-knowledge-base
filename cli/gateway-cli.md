---
domain: cli
topic: "Gateway CLI: openclaw gateway start, stop, status, force, and Daemon Commands"
type: reference
keywords:
  - openclaw gateway
  - gateway CLI
  - openclaw daemon
  - gateway start
  - gateway stop
  - gateway status
related:
  - gateway/gateway-runbook
  - cli/cli-overview
source: cli/gateway.md
---

The `openclaw gateway` command manages the Gateway process: start, stop, status, force restart, and more.

The Gateway is OpenClaw's WebSocket server (channels, nodes, sessions, hooks). Subcommands in this page live under `openclaw gateway …`.

    Local mDNS + wide-area DNS-SD setup.

    How OpenClaw advertises and finds gateways.

    Top-level gateway config keys.

## Run the Gateway

Run a local Gateway process:

```bash
openclaw gateway
```

Foreground alias:

```bash
openclaw gateway run
```

    - By default, the Gateway refuses to start unless `gateway.mode=local` is set in `~/.openclaw/openclaw.json`. Use `--allow-unconfigured` for ad-hoc/dev runs.
    - `openclaw onboard --mode local` and `openclaw setup` are expected to write `gateway.mode=local`. If the file exists but `gateway.mode` is missing, treat that as a broken or clobbered config and repair it instead of assuming local mode implicitly.
    - If the file exists and `gateway.mode` is missing, the Gateway treats that as suspicious config damage and refuses to "guess local" for you.
    - Binding beyond loopback without auth is blocked (safety guardrail).
    - `SIGUSR1` triggers an in-process restart when authorized (`commands.restart` is enabled by default; set `commands.restart: false` to block manual restart, while gateway tool/config apply/update remain allowed).
    - `SIGINT`/`SIGTERM` handlers stop the gateway process, but they don't restore any custom terminal state. If you wrap the CLI with a TUI or raw-mode input, restore the terminal before exit.

### Options

<ParamField path="--port <port>" type="number">
  WebSocket port (default comes from config/env; usually `18789`).
</ParamField>
<ParamField path="--bind <loopback|lan|tailnet|auto|custom>" type="string">
  Listener bind mode.
</ParamField>
<ParamField path="--auth <token|password>" type="string">
  Auth mode override.
</ParamField>
<ParamField path="--token <token>" type="string">
  Token override (also sets `OPENCLAW_GATEWAY_TOKEN` for the process).
</ParamField>
<ParamField path="--password <password>" type="string">
  Password override.
</ParamField>
<ParamField path="--password-file <path>" type="string">
  Read the gateway password from a file.
</ParamField>
<ParamField path="--tailscale <off|serve|funnel>" type="string">
  Expose the Gateway via Tailscale.
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  Reset Tailscale serve/funnel config on shutdown.
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  Allow gateway start without `gateway.mode=local` in config. Bypasses the startup guard for ad-hoc/dev bootstrap only; does not write or repair the config file.
</ParamField>
<ParamField path="--dev" type="boolean">
  Create a dev config + workspace if missing (skips BOOTSTRAP.md).
</ParamField>
<ParamField path="--reset" type="boolean">
  Reset dev config + credentials + sessions + workspace (requires `--dev`).
</ParamField>
<ParamField path="--force" type="boolean">
  Kill any existing listener on the selected port before starting.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Verbose logs.
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  Only show CLI backend logs in the console (and enable stdout/stderr).
</ParamField>
<ParamField path="--ws-log <auto|full|compact>" type="string" default="auto">
  Websocket log style.
</ParamField>
<ParamField path="--compact" type="boolean">
  Alias for `--ws-log compact`.
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  Log raw model stream events to jsonl.
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  Raw stream jsonl path.
</ParamField>

Inline `--password` can be exposed in local process listings. Prefer `--password-file`, env, or a SecretRef-backed `gateway.auth.password`.

### Startup profiling

- Set `OPENCLAW_GATEWAY_STARTUP_TRACE=1` to log phase timings during Gateway startup, including per-phase `eventLoopMax` delay and plugin lookup-table timings for installed-index, manifest registry, startup planning, and owner-map work.
- Set `OPENCLAW_DIAGNOSTICS=timeline` with `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` to write a best-effort JSONL startup diagnostics timeline for external QA harnesses. You can also enable the flag with `diagnostics.flags: ["timeline"]` in config; the path is still env-provided. Add `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` to include event-loop samples.
- Run `pnpm test:startup:gateway -- --runs 5 --warmup 1` to benchmark Gateway startup. The benchmark records first process output, `/healthz`, `/readyz`, startup trace timings, event-loop delay, and plugin lookup-table timing details.

## Query a running Gateway

All query commands use WebSocket RPC.

    - Default: human-readable (colored in TTY).
    - `--json`: machine-readable JSON (no styling/spinner).
    - `--no-color` (or `NO_COLOR=1`): disable ANSI while keeping human layout.

    - `--url <url>`: Gateway WebSocket URL.
    - `--token <token>`: Gateway token.
    - `--password <password>`: Gateway password.
    - `--timeout <ms>`: timeout/budget (varies per command).
    - `--expect-final`: wait for a "final" response (agent calls).

When you set `--url`, the CLI does not fall back to config or environment credentials. Pass `--token` or `--password` explicitly. Missing explicit credentials is an error.

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
```

The HTTP `/healthz` endpoint is a liveness probe: it returns once the server can answer HTTP. The HTTP `/readyz` endpoint is stricter and stays red while startup sidecars, channels, or configured hooks are still settling. Local or authenticated detailed readiness responses include an `eventLoop` diagnostic block with event-loop delay, event-loop utilization, CPU core ratio, and a `degraded` flag.

### `gateway usage-cost`

Fetch usage-cost summaries from session logs.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  Number of days to include.
</ParamField>

### `gateway stability`

Fetch the recent diagnostic stability recorder from a running Gateway.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  Maximum number of recent events to include (max `1000`).
</ParamField>
<ParamField path="--type <type>" type="string">
  Filter by diagnostic event type, such as `payload.large` or `diagnostic.memory.pressure`.
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  Include only events after a diagnostic sequence number.
</ParamField>
<ParamField path="--bundle [path]" type="string">
  Read a persisted stability bundle instead of calling the running Gateway. Use `--bundle latest` (or just `--bundle`) for the newest bundle under the state directory, or pass a bundle JSON path directly.
</ParamField>
<ParamField path="--export" type="boolean">
  Write a shareable support diagnostics zip instead of printing stability details.
</ParamField>
<ParamField path="--output <path>" type="string">
  Output path for `--export`.
</ParamField>

    - Records keep operational metadata: event names, counts, byte sizes, memory readings, queue/session state, channel/plugin names, and redacted session summaries. They do not keep chat text, webhook bodies, tool outputs, raw request or response bodies, tokens, cookies, secret values, hostnames, or raw session ids. Set `diagnostics.enabled: false` to disable the recorder entirely.
    - On fatal Gateway exits, shutdown timeouts, and restart startup failures, OpenClaw writes the same diagnostic snapshot to `~/.openclaw/logs/stability/openclaw-stability-*.json` when the recorder has events. Inspect the newest bundle with `openclaw gateway stability --bundle latest`; `--limit`, `--type`, and `--since-seq` also apply to bundle output.

### `gateway diagnostics export`

Write a local diagnostics zip that is designed to attach to bug reports. For the privacy model and bundle contents, see [Diagnostics Export](/gateway/diagnostics).

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  Output zip path. Defaults to a support export under the state directory.
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  Maximum sanitized log lines to include.
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  Maximum log bytes to inspect.
</ParamField>
<ParamField path="--url <url>" type="string">
  Gateway WebSocket URL for the health snapshot.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway token for the health snapshot.
</ParamField>
<P

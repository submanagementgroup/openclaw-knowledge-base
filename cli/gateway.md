---
domain: cli
topic: "CLI: openclaw gateway Commands"
type: reference
keywords:
  - openclaw gateway
  - gateway CLI
  - gateway commands
  - start gateway
  - gateway options
source: cli/gateway.md
---

The Gateway is OpenClaw's WebSocket server (channels, nodes, sessions, hooks). Subcommands in this page live under `openclaw gateway …`.

**Bonjour discovery:** Local mDNS + wide-area DNS-SD setup.
  

**Discovery overview:** How OpenClaw advertises and finds gateways.
  

**Configuration:** Top-level gateway config keys.
  

## Run the Gateway

Run a local Gateway process:

```bash
openclaw gateway
```

Foreground alias:

```bash
openclaw gateway run
```

### Startup behavior

- By default, the Gateway refuses to start unless `gateway.mode=local` is set in `~/.openclaw/openclaw.json`. Use `--allow-unconfigured` for ad-hoc/dev runs.
    - `openclaw onboard --mode local` and `openclaw setup` are expected to write `gateway.mode=local`. If the file exists but `gateway.mode` is missing, treat that as a broken or clobbered config and repair it instead of assuming local mode implicitly.
    - If the file exists and `gateway.mode` is missing, the Gateway treats that as suspicious config damage and refuses to "guess local" for you.
    - Binding beyond loopback without auth is blocked (safety guardrail).
    - `SIGUSR1` triggers an in-process restart when authorized (`commands.restart` is enabled by default; set `commands.restart: false` to block manual restart, while gateway tool/config apply/update remain allowed).
    - `SIGINT`/`SIGTERM` handlers stop the gateway process, but they don't restore any custom terminal state. If you wrap the CLI with a TUI or raw-mode input, restore the terminal before exit.

  ### Options

" type="number">
  WebSocket port (default comes from config/env; usually `18789`).

" type="string">
  Listener bind mode.

" type="string">
  Auth mode override.

" type="string">
  Token override (also sets `OPENCLAW_GATEWAY_TOKEN` for the process).

" type="string">
  Password override.

" type="string">
  Read the gateway password from a file.

" type="string">
  Expose the Gateway via Tailscale.


  Reset Tailscale serve/funnel config on shutdown.


  Allow gateway start without `gateway.mode=local` in config. Bypasses the startup guard for ad-hoc/dev bootstrap only; does not write or repair the config file.


  Create a dev config + workspace if missing (skips BOOTSTRAP.md).


  Reset dev config + credentials + sessions + workspace (requires `--dev`).


  Kill any existing listener on the selected port before starting.


  Verbose logs.


  Only show CLI backend logs in the console (and enable stdout/stderr).

" type="string" default="auto">
  Websocket log style.


  Alias for `--ws-log compact`.


  Log raw model stream events to jsonl.

" type="string">
  Raw stream jsonl path.


## Restart the Gateway

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
```

`openclaw gateway restart --safe` asks the running Gateway to preflight active OpenClaw work before restarting. If queued operations, reply delivery, embedded runs, or task runs are active, the Gateway reports the blockers, coalesces duplicate safe restart requests, and restarts once the active work drains. Plain `restart` keeps the existing service-manager behavior for compatibility. Use `--force` only when you explicitly want the immediate override path.

`openclaw gateway restart --safe --skip-deferral` runs the same OpenClaw-aware coordinated restart as `--safe`, but bypasses the active-work deferral gate so the Gateway emits the restart immediately even when blockers are reported. Use it as the operator escape hatch when a deferral has been pinned by a stuck task run and `--safe` alone would wait indefinitely. `--skip-deferral` requires `--safe`.

> **Note:** Inline `--password` can be exposed in local process listings. Prefer `--password-file`, env, or a SecretRef-backed `gateway.auth.password`.


### Startup profiling

- Set `OPENCLAW_GATEWAY_STARTUP_TRACE=1` to log phase timings during Gateway startup, including per-phase `eventLoopMax` delay and plugin lookup-table timings for installed-index, manifest registry, startup planning, and owner-map work.
- Set `OPENCLAW_DIAGNOSTICS=timeline` with `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` to write a best-effort JSONL startup diagnostics timeline for external QA harnesses. You can also enable the flag with `diagnostics.flags: ["timeline"]` in config; the path is still env-provided. Add `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` to include event-loop samples.
- Run `pnpm test:startup:gateway -- --runs 5 --warmup 1` to benchmark Gateway startup. The benchmark records first process output, `/healthz`, `/readyz`, startup trace timings, event-loop delay, and plugin lookup-table timing details.

## Query a running Gateway

All query commands use WebSocket RPC.

**Output modes:**

- Default: human-readable (colored in TTY).
    - `--json`: machine-readable JSON (no styling/spinner).
    - `--no-color` (or `NO_COLOR=1`): disable ANSI while keeping human layout.

  
**Shared options:**

- `--url <url>`: Gateway WebSocket URL.
    - `--token <token>`: Gateway token.
    - `--password <password>`: Gateway password.
    - `--timeout <ms>`: timeout/budget (varies per command).
    - `--expect-final`: wait for a "final" response (agent calls).

  
> **Note:** When you set `--url`, the CLI does not fall back to config or environment credentials. Pass `--token` or `--password` explicitly. Missing explicit credentials is an error.


### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
```

The HTTP `/healthz` endpoint is a liveness probe: it returns once the server can answer HTTP. The HTTP `/readyz` endpoint is stricter and stays red while startup plugin sidecars, channels, or configured hooks are still settling. Local or authenticated detailed readiness responses include an `eventLoop` diagnostic block with event-loop delay, event-loop utilization, CPU core ratio, and a `degraded` flag.

### `gateway usage-cost`

Fetch usage-cost summaries from session logs.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --json
```

" type="number" default="30">
  Number of days to include.


### `gateway stability`

Fetch the recent diagnostic stability recorder from a running Gateway.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

" type="number" default="25">
  Maximum number of recent events to include (max `1000`).

" type="string">
  Filter by diagnostic event type, such as `payload.large` or `diagnostic.memory.pressure`.

" type="number">
  Include only events after a diagnostic sequence number.


  Read a persisted stability bundle instead of calling the running Gateway. Use `--bundle latest` (or just `--bundle`) for the newest bundle under the state directory, or pass a bundle JSON path directly.


  Write a shareable support diagnostics zip instead of printing stability details.

" type="string">
  Output path for `--export`.


### Privacy and bundle behavior

- Records keep operational metadata: event names, counts, byte sizes, memory readings, queue/session state, channel/plugin names, and redacted session summaries. They do not keep chat text, webhook bodies, tool outputs, raw request or response bodies, tokens, cookies, secret values, hostnames, or raw session ids. Set `diagnostics.enabled: false` to disable the recorder entirely.
    - On fatal Gateway exits, shutdown timeouts, and restart startup failures, OpenClaw writes the same diagnostic snapshot to `~/.openclaw/logs/stability/openclaw-stability-*.json` when the recorder has events. Inspect the newest bundle with `openclaw gateway stability --bundle latest`; `--limit`, `--type`, and `--since-seq` also apply to bundle output.

  ### `gateway diagnostics export`

Write a local diagnostics zip that is designed to attach to bug reports. For the privacy model and bundle contents, see [Diagnostics Export](/gateway/diagnostics).

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

" type="string">
  Output zip path. Defaults to a support export under the state directory.

" type="number" default="5000">
  Maximum sanitized log lines to include.

" type="number" default="1000000">
  Maximum log bytes to inspect.

" type="string">
  Gateway WebSocket URL for the health snapshot.

" type="string">
  Gateway token for the health snapshot.

" type="string">
  Gateway password for the health snapshot.

" type="number" default="3000">
  Status/health snapshot timeout.


  Skip persisted stability bundle lookup.


  Print the written path, size, and manifest as JSON.


The export contains a manifest, a Markdown summary, config shape, sanitized config details, sanitized log summaries, sanitized Gateway status/health snapshots, and the newest stability bundle when one exists.

It is meant to be shared. It keeps operational details that help debugging, such as safe OpenClaw log fields, subsystem names, status codes, durations, configured modes, ports, plugin ids, provider ids, non-secret feature settings, and redacted operational log messages. It omits or redacts chat text, webhook bodies, tool outputs, credentials, cookies, account/message identifiers, prompt/instruction text, hostnames, and secret values. When a LogTape-style message looks like user/chat/tool payload text, the export keeps only that a message was omitted plus its byte count.

### `gateway status`

`gateway status` shows the Gateway service (launchd/systemd/schtasks) plus an optional probe of connectivity/auth capability.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

" type="string">
  Add an explicit probe target. Configured remote + localhost are still probed.

" type="string">
  Token auth for the probe.

" type="string">
  Password auth for the probe.

" type="number" default="10000">
  Probe timeout.


  Skip the connectivity probe (service-only view).


  Scan system-level services too.


  Upgrade the default connectivity probe to a read probe and exit non-zero when that read probe fails. Cannot be combined with `--no-probe`.


### Status semantics

- `gateway status` stays available for diagnostics even when the local CLI config is missing or invalid.
    - Default `gateway status` proves service state, WebSocket connect, and the auth capability visible at handshake time. It does not prove read/write/admin operations.
    - Diagnostic probes are non-mutating for first-time device auth: they reuse an existing cached device token when one exists, but they do not create a new CLI device identity or read-only device pairing record just to check status.
    - `gateway status` resolves configured auth SecretRefs for probe auth when possible.
    - If a required auth SecretRef is unresolved in this command path, `gateway status --json` reports `rpc.authWarning` when probe connectivity/auth fails; pass `--token`/`--password` explicitly or resolve the secret source first.
    - If the probe succeeds, unresolved auth-ref warnings are suppressed to avoid false positives.
    - Use `--require-rpc` in scripts and automation when a listening service is not enough and you need read-scope RPC calls to be healthy too.
    - `--deep` adds a best-effort scan for extra launchd/systemd/schtasks installs. When multiple gateway-like services are detected, human output prints cleanup hints and warns that most setups should run one gateway per machine.
    - `--deep` also reports a recent Gateway supervisor restart handoff when the service process exited cleanly for an external supervisor restart.
    - `--deep` runs config validation in plugin-aware mode (`pluginValidation: "full"`) and surfaces configured plugin manifest warnings (for example missing channel config metadata) so install and update smoke checks catch them. Default `gateway status` keeps the fast read-only path that skips plugin validation.
    - Human output includes the resolved file log path plus the CLI-vs-service config paths/validity snapshot to help diagnose profile or state-dir drift.

  ### Linux systemd auth-drift checks

- On Linux systemd installs, service auth drift checks read both `Environment=` and `EnvironmentFile=` values from the unit (including `%h`, quoted paths, multiple files, and optional `-` files).
    - Drift checks resolve `gateway.auth.token` SecretRefs using merged runtime env (service command env first, then process env fallback).
    - If token auth is not effectively active (explicit `gateway.auth.mode` of `password`/`none`/`trusted-proxy`, or mode unset where password can win and no token candidate can win), token-drift checks skip config token resolution.

  ### `gateway probe`

`gateway probe` is the "debug everything" command. It always probes:

- your configured remote gateway (if set), and
- localhost (loopback) **even if remote is configured**.

If you pass `--url`, that explicit target is added ahead of both. Human output labels the targets as:

- `URL (explicit)`
- `Remote (configured)` or `Remote (configured, inactive)`
- `Local loopback`

> **Note:** If multiple gateways are reachable, it prints all of them. Multiple gateways are supported when you use isolated profiles/ports (e.g., a rescue bot), but most installs still run a single gateway.


```bash
openclaw gateway probe
openclaw gateway probe --json
```

### Interpretation

- `Reachable: yes` means at least one target accepted a WebSocket connect.
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` reports what the probe could prove about auth. It is separate from reachability.
    - `Read probe: ok` means read-scope detail RPC calls (`health`/`status`/`system-presence`/`config.get`) also succeeded.
    - `Read probe: limited - m
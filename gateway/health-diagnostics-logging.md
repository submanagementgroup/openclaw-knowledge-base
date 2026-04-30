---
domain: gateway
topic: "Gateway Health, Diagnostics, and Logging: openclaw doctor, health, and logs"
type: procedure
keywords:
  - health check
  - diagnostics
  - logging
  - openclaw doctor
  - openclaw logs
  - openclaw health
  - log level
related:
  - gateway/troubleshooting
  - gateway/gateway-runbook
source:
  - gateway/health.md
  - gateway/diagnostics.md
  - gateway/logging.md
  - gateway/doctor.md
---

Gateway health, diagnostics, and logging tools for monitoring and debugging OpenClaw.

## Health Check

```bash
openclaw health          # quick health status
openclaw gateway status  # detailed gateway status
openclaw status          # overall system status
```

## Doctor (Deep Diagnostics)

`openclaw doctor` performs a comprehensive health check:

```bash
openclaw doctor           # full interactive diagnostics
openclaw doctor --json    # machine-readable output
```

Doctor checks: Gateway connectivity, channel status, model availability, workspace permissions, secrets, and daemon status.

`openclaw doctor` is the repair + migration tool for OpenClaw. It fixes stale config/state, checks health, and provides actionable repair steps.

## Quick start

```bash
openclaw doctor
```

### Headless and automation modes

    ```bash
    openclaw doctor --yes
    ```

    Accept defaults without prompting (including restart/service/sandbox repair steps when applicable).

    ```bash
    openclaw doctor --repair
    ```

    Apply recommended repairs without prompting (repairs + restarts where safe).

    ```bash
    openclaw doctor --repair --force
    ```

    Apply aggressive repairs too (overwrites custom supervisor configs).

    ```bash
    openclaw doctor --non-interactive
    ```

    Run without prompts and only apply safe migrations (config normalization + on-disk state moves). Skips restart/service/sandbox actions that require human confirmation. Legacy state migrations run automatically when detected.

    ```bash
    openclaw doctor --deep
    ```

    Scan system services for extra gateway installs (launchd/systemd/schtasks).

If you want to review changes before writing, open the config file first:

```bash
cat ~/.openclaw/openclaw.json
```

## What it does (summary)

    - Optional pre-flight update for git installs (interactive only).
    - UI protocol freshness check (rebuilds Control UI when the protocol schema is newer).
    - Health check + restart prompt.
    - Skills status summary (eligible/missing/blocked) and plugin status.

    - Config normalization for legacy values.
    - Talk config migration from legacy flat `talk.*` fields into `talk.provider` + `talk.providers.<provider>`.
    - Browser migration checks for legacy Chrome extension configs and Chrome MCP readiness.
    - OpenCode provider override warnings (`models.providers.opencode` / `models.providers.opencode-go`).
    - Codex OAuth shadowing warnings (`models.providers.openai-codex`).
    - OAuth TLS prerequisites check for OpenAI Codex OAuth profiles.
    - Legacy on-disk state migration (sessions/agent dir/WhatsApp auth).
    - Legacy plugin manifest contract key migration (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
    - Legacy cron store migration (`jobId`, `schedule.cron`, top-level delivery/payload fields, payload `provider`, simple `notify: true` webhook fallback jobs).
    - Legacy agent runtime-policy migration to `agents.defaults.agentRuntime` and `agents.list[].agentRuntime`.
    - Stale plugin config cleanup when plugins are enabled; when `plugins.enabled=false`, stale plugin references are treated as inert containment config and are preserved.

    - Session lock file inspection and stale lock cleanup.
    - Session transcript repair for duplicated prompt-rewrite branches created by affected 2026.4.24 builds.
    - State integrity and permissions checks (sessions, t

## Logging

```bash
openclaw logs --follow          # tail logs
openclaw logs --lines 200       # last 200 lines
openclaw logs --level debug     # set log level
```

Log file: `~/.openclaw/openclaw.log` (default)

Configure log level in `openclaw.json`:
```json5
{ logging: { level: "info" } }   // "error" | "warn" | "info" | "debug" | "trace"
```

# Logging

For a user-facing overview (CLI + Control UI + config), see [/logging](/logging).

OpenClaw has two log “surfaces”:

- **Console output** (what you see in the terminal / Debug UI).
- **File logs** (JSON lines) written by the gateway logger.

## File-based logger

- Default rolling log file is under `/tmp/openclaw/` (one file per day): `openclaw-YYYY-MM-DD.log`
  - Date uses the gateway host's local timezone.
- Active log files rotate at `logging.maxFileBytes` (default: 100 MB), keeping
  up to five numbered archives and continuing to write a fresh active file.
- The log file path and level can be configured via `~/.openclaw/openclaw.json`:
  - `logging.file`
  - `logging.level`

The file format is one JSON object per line.

The Control UI Logs tab tails this file via the gateway (`logs.tail`).
CLI can do the same:

```bash
openclaw logs --follow
```

**Verbose vs. log levels**

- **File logs** are controlled exclusively by `logging.level`.
- `--verbose` only affects **console verbosity** (and WS log style); it does **not**
  raise the file log level.
- To capture verbose-only details in file logs, set `logging.level` to `debug` or
  `trace`.

## Console capture

The CLI captures `console.log/info/warn/error/debug/trace` and writes them to file logs,
while still printing to stdout/stderr.

You can tune console verbosity independently via:

- `logging.consoleLevel` (default `info`)
- `logging.consoleStyle` (`pretty` | `compact` | `json`)

## Redaction

OpenClaw can mask sensitive tokens before log or transcript output leaves the
process. This logging redaction policy is applied at console, file-log, OTLP
log-record, and session transcript text sinks, so matching secret values are
masked before JSONL lines or messages are written to disk.

- `logging.redactSensitive`: `off` | `tools` (default: `tools`)
- `logging.redactPatterns`: array of regex strings (overrides defaults)
  - Use raw regex strings (auto `gi`), or `/pattern/flags` if you need custom flags.
 

## Diagnostics Flags

OpenClaw can create a local diagnostics zip for bug reports. It combines
sanitized Gateway status, health, logs, config shape, and recent payload-free
stability events.

Treat diagnostics bundles like secrets until you have reviewed them. They are
designed to omit or redact payloads and credentials, but they still summarize
local Gateway logs and host-level runtime state.

## Quick start

```bash
openclaw gateway diagnostics export
```

The command prints the written zip path. To choose a path:

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

For automation:

```bash
openclaw gateway diagnostics export --json
```

## Chat command

Owners can use `/diagnostics [note]` in chat to request a local Gateway export.
Use this when the bug happened in a real conversation and you want one
copy-pasteable report for support:

1. Send `/diagnostics` in the conversation where you noticed the problem. Add a
   short note if it helps, for example `/diagnostics bad tool choice`.
2. OpenClaw sends the diagnostics preamble and asks for one explicit exec
   approval. The approval runs `openclaw gateway diagnostics export --json`.
   Do not approve diagnostics through an allow-all rule.
3. After approval, OpenClaw replies with a pasteable report containing the local
   bundle path, manifest summary, privacy notes, and relevant session ids.

In group chats, an owner can still run `/diagnostics`, but OpenClaw does not
post the diagnostic details back into the shared chat. It sends the preamble,
approval prompts, Gateway export result, and Codex session/thread breakdown to
the owner through the private approval route. The group only gets a short notice
that the diagnostics flow was sent privately. If OpenClaw cannot find a private
owner route, the command fails closed and asks the owner to run it from a DM.

When the active OpenClaw session is using the native OpenAI Codex harness,
the same exec approval also covers an OpenAI feedback upload for the Codex

## Health Endpoint

GET `http://127.0.0.1:18789/health` returns JSON with gateway health details. Use this for uptime monitoring.

Short guide to verify channel connectivity without guessing.

## Quick checks

- `openclaw status` — local summary: gateway reachability/mode, update hint, linked channel auth age, sessions + recent activity.
- `openclaw status --all` — full local diagnosis (read-only, color, safe to paste for debugging).
- `openclaw status --deep` — asks the running gateway for a live health probe (`health` with `probe:true`), including per-account channel probes when supported.
- `openclaw health` — asks the running gateway for its health snapshot (WS-only; no direct channel sockets from the CLI).
- `openclaw health --verbose` — forces a live health probe and prints gateway connection details.
- `openclaw health --json` — machine-readable health snapshot output.
- Send `/status` as a standalone message in WhatsApp/WebChat to get a status reply without invoking the agent.
- Logs: tail `/tmp/openclaw/openclaw-*.log` and filter for `web-heartbeat`, `web-reconnect`, `web-auto-reply`, `web-inbound`.

## Deep diagnostics

- Creds on disk: `ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (mtime should be recent).
- Session store: `ls -l ~/.openclaw/agents/<agentId>/sessions/sessions.json` (path can be overridden in config). Count and recent recipients are surfaced via `status`.
- Relink flow: `openclaw channels logout && openclaw channels login --verbose` when status codes 409–515 or `loggedOut` appear in logs. (Note: the QR login flow auto-restarts once for status 515 after pairing

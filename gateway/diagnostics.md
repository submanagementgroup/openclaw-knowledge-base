---
domain: gateway
topic: "Gateway Diagnostics"
type: reference
keywords:
  - diagnostics
  - gateway diagnostics
  - debug mode
  - tracing
  - verbose logging
source: gateway/diagnostics.md
---

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
runtime threads OpenClaw knows about. That upload is separate from the local
Gateway zip and appears only for Codex harness sessions. Before approval, the
prompt explains that approving diagnostics will also send Codex feedback, but it
does not list Codex session or thread ids. After approval, the chat reply lists
the channels, OpenClaw session ids, Codex thread ids, and local resume commands
for the threads that were sent to OpenAI servers. If you deny or ignore the
approval, OpenClaw does not run the export, does not send Codex feedback, and
does not print the Codex ids.

That makes the common Codex debugging loop short: notice the bad behavior in
Telegram, Discord, or another channel, run `/diagnostics`, approve once, share
the report with support, then run the printed `codex resume <thread-id>` command
locally if you want to inspect the native Codex thread yourself. See
[Codex harness](/plugins/codex-harness#inspect-codex-threads-locally) for
that inspection workflow.

## What the export contains

The zip includes:

- `summary.md`: human-readable overview for support.
- `diagnostics.json`: machine-readable summary of config, logs, status, health,
  and stability data.
- `manifest.json`: export metadata and file list.
- Sanitized config shape and non-secret config details.
- Sanitized log summaries and recent redacted log lines.
- Best-effort Gateway status and health snapshots.
- `stability/latest.json`: newest persisted stability bundle, when available.

The export is useful even when the Gateway is unhealthy. If the Gateway cannot
answer status or health requests, the local logs, config shape, and latest
stability bundle are still collected when available.

## Privacy model

Diagnostics are designed to be shareable. The export keeps operational data
that helps debugging, such as:

- subsystem names, plugin ids, provider ids, channel ids, and configured modes
- status codes, durations, byte counts, queue state, and memory readings
- sanitized log metadata and redacted operational messages
- config shape and non-secret feature settings

The export omits or redacts:

- chat text, prompts, instructions, webhook bodies, and tool outputs
- credentials, API keys, tokens, cookies, and secret values
- raw request or response bodies
- account ids, message ids, raw session ids, hostnames, and local usernames

When a log message looks like user, chat, prompt, or tool payload text, the
export keeps only that a message was omitted and the byte count.

## Stability recorder

The Gateway records a bounded, payload-free stability stream by default when
diagnostics are enabled. It is for operational facts, not content.

The same diagnostic heartbeat records liveness samples when the Gateway keeps
running but the Node.js event loop or CPU looks saturated. These
`diagnostic.liveness.warning` events include event-loop delay, event-loop
utilization, CPU-core ratio, active/waiting/queued session counts, the current
startup/runtime phase when known, recent phase spans, and bounded active/queued
work labels. Idle samples stay in telemetry at `info` level. Liveness samples
become Gateway warnings only when work is waiting or queued, or when active work
overlaps with sustained event-loop delay. Transient max-delay spikes during
otherwise healthy background work stay in debug logs. They do not restart the
Gateway by themselves.

Startup phases also emit `diagnostic.phase.completed` events with wall-clock and
CPU timing. Stalled embedded-run diagnostics mark `terminalProgressStale=true`
when the last bridge progress looked terminal, such as a raw response item or
response completion event, but the Gateway still considers the embedded run
active.

Inspect the live recorder:

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

Inspect the newest persisted stability bundle after a fatal exit, shutdown
timeout, or restart startup failure:

```bash
openclaw gateway stability --bundle latest
```

Create a diagnostics zip from the newest persisted bundle:

```bash
openclaw gateway stability --bundle latest --export
```

Persisted bundles live under `~/.openclaw/logs/stability/` when events exist.

## Useful options

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

- `--output <path>`: write to a specific zip path.
- `--log-lines <count>`: maximum sanitized log lines to include.
- `--log-bytes <bytes>`: maximum log bytes to inspect.
- `--url <url>`: Gateway WebSocket URL for status and health snapshots.
- `--token <token>`: Gateway token for status and health snapshots.
- `--password <password>`: Gateway password for status and health snapshots.
- `--timeout <ms>`: status and health snapshot timeout.
- `--no-stability-bundle`: skip persisted stability bundle lookup.
- `--json`: print machine-readable export metadata.

## Disable diagnostics

Diagnostics are enabled by default. To disable the stability recorder and
diagnostic event collection:

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

Disabling diagnostics reduces bug-report detail. It does not affect normal
Gateway logging.

## Related

- [Health checks](/gateway/health)
- [Gateway CLI](/cli/gateway#gateway-diagnostics-export)
- [Gateway protocol](/gateway/protocol#system-and-identity)
- [Logging](/logging)
- [OpenTelemetry export](/gateway/opentelemetry) — separate flow for streaming diagnostics to a collector


## Secure File Operations

OpenClaw uses [`@openclaw/fs-safe`](https://github.com/openclaw/fs-safe) for security-sensitive local file operations: root-bounded reads/writes, atomic replacement, archive extraction, temp workspaces, JSON state, and secret-file handling.

The goal is a consistent **library guardrail** for trusted OpenClaw code that receives untrusted path names. It is not a sandbox. Host filesystem permissions, OS users, containers, and the agent/tool policy still define the real blast radius.

## Default: no Python helper

OpenClaw defaults the fs-safe POSIX Python helper to **off**.

Why:

- the gateway should not spawn a persistent Python sidecar unless an operator opted into it;
- many installs do not need the extra parent-directory mutation hardening;
- disabling Python keeps package/runtime behavior more predictable across desktop, Docker, CI, and bundled app environments.

OpenClaw only changes the default. If you explicitly set a mode, fs-safe honors it:

```bash
# Default OpenClaw behavior: Node-only fs-safe fallbacks.
OPENCLAW_FS_SAFE_PYTHON_MODE=off

# Opt into the helper when available, falling back if unavailable.
OPENCLAW_FS_SAFE_PYTHON_MODE=auto

# Fail closed if the helper cannot start.
OPENCLAW_FS_SAFE_PYTHON_MODE=require

# Optional explicit interpreter.
OPENCLAW_FS_SAFE_PYTHON=/usr/bin/python3
```

The generic fs-safe names also work: `FS_SAFE_PYTHON_MODE` and `FS_SAFE_PYTHON`.

## What stays protected without Python

With the helper off, OpenClaw still uses fs-safe's Node paths for:

- rejecting relative-path escapes such as `..`, absolute paths, and path separators where only names are allowed;
- resolving operations through a trusted root handle instead of ad-hoc `path.resolve(...).startsWith(...)` checks;
- refusing symlink and hardlink patterns on APIs that require that policy;
- opening files with identity checks where the API returns or consumes file contents;
- atomic sibling-temp writes for state/config files;
- byte limits for reads and archive extraction;
- private modes for secrets and state files where the API requires them.

These protections cover the normal OpenClaw threat model: trusted gateway code handling untrusted model/plugin/channel path input inside a single trusted operator boundary.

## What Python adds

On POSIX, fs-safe's optional helper keeps one persistent Python process and uses fd-relative filesystem operations for parent-directory mutations such as rename, remove, mkdir, stat/list, and some write paths.

That narrows same-UID race windows where another process can swap a parent directory between validation and mutation. It is defense in depth for hosts where untrusted local processes can modify the same directories OpenClaw is operating in.

If your deployment has that risk and Python is guaranteed to exist, use:

```bash
OPENCLAW_FS_SAFE_PYTHON_MODE=require
```

Use `require` rather than `auto` when the helper is part of your security posture; `auto` intentionally falls back to Node-only behavior if the helper is unavailable.

## Plugin and core guidance

- Plugin-facing file access should go through `openclaw/plugin-sdk/*` helpers, not raw `fs`, when a path comes from a message, model output, config, or plugin input.
- Core code should use the local fs-safe wrappers under `src/infra/*` so OpenClaw's process policy is applied consistently.
- Archive extraction should use the fs-safe archive helpers with explicit size, entry-count, link, and destination limits.
- Secrets should use OpenClaw secret helpers or fs-safe secret/private-state helpers; do not hand-roll mode checks around `fs.writeFile`.
- If you need hostile local-user isolation, do not rely on fs-safe alone. Run separate gateways under separate OS users/hosts or use sandboxing.

Related: [Security](/gateway/security), [Sandboxing](/gateway/sandboxing), [Exec approvals](/tools/exec-approvals), [Secrets](/gateway/secrets).


## Index

> **Note:** **Personal assistant trust model.** This guidance assume
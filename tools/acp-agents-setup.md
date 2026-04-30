---
domain: tools
topic: "ACP Agents Setup: Configuration, Session Binding, and Delivery Model"
type: procedure
keywords:
  - ACP setup
  - ACP config
  - ACP session binding
  - ACP delivery
  - acpx harness
related:
  - tools/acp-agents
  - tools/subagents
source:
  - tools/acp-agents-setup.md
  - tools/acp-agents.md
---

ACP setup, configuration, and advanced features.

ermission profile that can proceed headlessly.

OpenClaw plugin tools and built-in OpenClaw tools are **not** exposed to
ACP harnesses by default. Enable the explicit MCP bridges in
[ACP agents — setup](/tools/acp-agents-setup) only when the harness
should call those tools directly.

## Supported harness targets

With the bundled `acpx` backend, use these harness ids as `/acp spawn <id>`
or `sessions_spawn({ runtime: "acp", agentId: "<id>" })` targets:

| Harness id | Typical backend                                | Notes                                                                               |
| ---------- | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `claude`   | Claude Code ACP adapter                        | Requires Claude Code auth on the host.                                              |
| `codex`    | Codex ACP adapter                              | Explicit ACP fallback only when native `/codex` is unavailable or ACP is requested. |
| `copilot`  | GitHub Copilot ACP adapter                     | Requires Copilot CLI/runtime auth.                                                  |
| `cursor`   | Cursor CLI ACP (`cursor-agent acp`)            | Override the acpx command if a local install exposes a different ACP entrypoint.    |
| `droid`    | Factory Droid CLI                              | Requires Factory/Droid auth or `FACTORY_API_KEY` in the harness environment.        |
| `gemini`   | Gemini CLI ACP adapter                         | Requires Gemini CLI auth or API key setup.                                          |
| `iflow`    | iFlow CLI                                      | Adapter availability and model control depend on the installed CLI.                 |
| `kilocode` | Kilo Code CLI                                  | Adapter availability and model control depend on the installed CLI.                 |
| `kimi`     | Kimi/Moonshot CLI                              | Requires Kimi/Moonshot auth on the host.                                            |
| `kiro`     | Kiro CLI                                       | Adapter availability and model control depend on the installed CLI.                 |
| `opencode` | OpenCode ACP adapter                           | Requires OpenCode CLI/provider auth.                                                |
| `openclaw` | OpenClaw Gateway bridge through `openclaw acp` | Lets an ACP-aware harness talk back to an OpenClaw Gateway session.                 |
| `pi`       | Pi/embedded OpenClaw runtime                   | Used for OpenClaw-native harness experiments.                                       |
| `qwen`     | Qwen Code / Qwen CLI                           | Requires Qwen-compatible auth on the host.                                          |

Custom acpx agent aliases can be configured in acpx itself, but OpenClaw
policy still checks `acp.allowedAgents` and any
`agents.list[].runtime.acp.agent` mapping before dispatch.

## Operator runbook

Quick `/acp` flow from chat:

    `/acp spawn claude --bind here`,
    `/acp spawn gemini --mode persistent --thread auto`, or explicit
    `/acp spawn codex --bind here`.

    Continue in the bound conversation or thread (or target the session
    key explicitly).

    `/acp status`

    `/acp model <provider/model>`,
    `/acp permissions <profile>`,
    `/acp timeout <seconds>`.

    Without replacing context: `/acp steer tighten logging and continue`.

    `/acp cancel` (current turn) or `/acp close` (session + bindings).

    - Spawn creates or resumes an ACP runtime session, records ACP metadata in the OpenClaw session store, and may create a background task when the run is parent-owned.
    - Parent-owned ACP sessions are treated as background work even when the runtime session is persistent; completion and cross-surface delivery go through the parent task notifier rather than acting like a normal user-facing chat session.
    - Task maintenance closes terminal or orphaned parent-owned one-shot ACP sessions. Persistent ACP sessions are preserved while an active conversation binding remains; stale persistent sessions without an active binding are closed so they cannot be silently resumed after the owning task is done or its task record is gone.
    - Bound follow-up messages go directly to the ACP session until the binding is closed, unfocused, reset, or expired.
    - Gateway commands stay local. `/acp ...`, `/status`, and `/unfocus` are never sent as normal prompt text to a bound ACP harness.
    - `cancel` aborts the active turn when the backend supports cancellation; it does not delete the binding or session metadata.
    - `close` ends the ACP session from OpenClaw's point of view and removes the binding. A harness may still keep its own upstream history if it supports resume.
    - Idle runtime workers are eligible for cleanup after `acp.runtime.ttlMinutes`; stored session metadata remains available for `/acp sessions`.

    Natural-language triggers that should route to the **native Codex
    plugin** when it is enabled:

    - "Bind this Discord channel to Codex."
    - "Attach this chat to Codex thread `<id>`."
    - "Show Codex threads, then bind this one."

    Native Codex conversation binding is the default chat-control path.
    OpenClaw dynamic tools still execute through OpenClaw, while
    Codex-native tools such as shell/apply-patch execute inside Codex.
    For Codex-native tool events, OpenClaw injects a per-turn native
    hook relay so plugin hooks can block `before_tool_call`, observe
    `after_tool_call`, and route Codex `PermissionRequest` events
    through OpenClaw approvals. Codex `Stop` hooks are relayed to
    OpenClaw `before_agent_finalize`, where plugins can request one more
    model pass before Codex finalizes its answer. The relay remains
    deliberately conservative: it does not mutate Codex-native tool
    arguments or rewrite Codex thread r

## ACP Setup Guide

For the overview, operator runbook, and concepts, see [ACP agents](/tools/acp-agents).

The sections below cover acpx harness config, plugin setup for the MCP bridges, and permission configuration.

Use this page only when you are setting up the ACP/acpx route. For native Codex
app-server runtime config, use [Codex harness](/plugins/codex-harness). For
OpenAI API keys or Codex OAuth model-provider config, use
[OpenAI](/providers/openai).

Codex has two OpenClaw routes:

| Route                      | Config/command                                         | Setup page                              |
| -------------------------- | ------------------------------------------------------ | --------------------------------------- |
| Native Codex app-server    | `/codex ...`, `agentRuntime.id: "codex"`               | [Codex harness](/plugins/codex-harness) |
| Explicit Codex ACP adapter | `/acp spawn codex`, `runtime: "acp", agentId: "codex"` | This page                               |

Prefer the native route unless you explicitly need ACP/acpx behavior.

## acpx harness support (current)

Current acpx built-in harness aliases:

- `claude`
- `codex`
- `copilot`
- `cursor` (Cursor CLI: `cursor-agent acp`)
- `droid`
- `gemini`
- `iflow`
- `kilocode`
- `kimi`
- `kiro`
- `openclaw`
- `opencode`
- `pi`
- `qwen`

When OpenClaw uses the acpx backend, prefer these values for `agentId` unless your acpx config defines custom agent aliases.
If your local Cursor install still exposes ACP as `agent acp`, override the `cursor` agent command in your acpx config instead of changing the built-in default.

Direct acpx CLI usage can also target arbitrary adapters via `--agent <command>`, but that raw escape hatch is an acpx CLI feature (not the normal OpenClaw `agentId` path).

Model control is adapter-capability dependent. Codex ACP model refs are
normalized by OpenClaw before startup. Other harnesses need ACP `models` plus
`session/set_model` support; if a harness exposes neither that ACP capability
nor its own startup model flag, OpenClaw/acpx cannot force a model selection.

## Required config

Core ACP baseline:

```json5
{
  acp: {
    enabled: true,
    // Optional. Default is true; set false to pause ACP dispatch while keeping /acp controls.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "pi",
      "qwen",
    ],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },
}
```

Thread binding config is channel-adapter specific. Example for Discord:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        spawnAcpSessions: true,
      },
    },
  },
}
```

If thread-bound ACP spawn does not work, verify the adapter feature flag first:

- Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

Current-conversation binds do not require child-thread creation. They require an active conversation context and a channel adapter that exposes ACP conversation bindings.

See [Configuration Reference](/gateway/configuration-reference).

## Plugin setup for acpx backend

Fresh installs ship the bundled `acpx` runtime plugin enabled by default, so ACP
usually works without a manual plugin install step.

Start with:

```text
/acp doctor
```

If you disabled `acpx`, denied it via `plugins.allow` / `plugins.deny`, or want
to switch to a local development checkout, use the explicit plugin path:

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

Local workspace install during development:

```bash
openclaw plugins

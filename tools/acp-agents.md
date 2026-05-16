---
domain: tools
topic: "ACP Agent Protocol: Agent-to-Agent Communication"
type: concept
keywords:
  - ACP
  - agent protocol
  - A2A
  - agent communication
  - ACP agents
  - multi-agent ACP
source: tools/acp-agents.md
---

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/) sessions
let OpenClaw run external coding harnesses (for example Pi, Claude Code,
Cursor, Copilot, Droid, OpenClaw ACP, OpenCode, Gemini CLI, and other
supported ACPX harnesses) through an ACP backend plugin.

Each ACP session spawn is tracked as a [background task](/automation/tasks).

> **Note:** **ACP is the external-harness path, not the default Codex path.** The
native Codex app-server plugin owns `/codex ...` controls and the default
`openai/gpt-*` embedded runtime for agent turns; ACP owns
`/acp ...` controls and `sessions_spawn({ runtime: "acp" })` sessions.

If you want Codex or Claude Code to connect as an external MCP client
directly to existing OpenClaw channel conversations, use
[`openclaw mcp serve`](/cli/mcp) instead of ACP.


## Which page do I want?

| You want to…                                                                                    | Use this                              | Notes                                                                                                                                                                                         |
| ----------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bind or control Codex in the current conversation                                               | `/codex bind`, `/codex threads`       | Native Codex app-server path when the `codex` plugin is enabled; includes bound chat replies, image forwarding, model/fast/permissions, stop, and steer controls. ACP is an explicit fallback |
| Run Claude Code, Gemini CLI, explicit Codex ACP, or another external harness _through_ OpenClaw | This page                             | Chat-bound sessions, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, background tasks, runtime controls                                                                                   |
| Expose an OpenClaw Gateway session _as_ an ACP server for an editor or client                   | [`openclaw acp`](/cli/acp)            | Bridge mode. IDE/client talks ACP to OpenClaw over stdio/WebSocket                                                                                                                            |
| Reuse a local AI CLI as a text-only fallback model                                              | [CLI Backends](/gateway/cli-backends) | Not ACP. No OpenClaw tools, no ACP controls, no harness runtime                                                                                                                               |

## Does this work out of the box?

Yes, after installing the official ACP runtime plugin:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Source checkouts can use the local `extensions/acpx` workspace plugin after
`pnpm install`. Run `/acp doctor` for a readiness check.

OpenClaw only teaches agents about ACP spawning when ACP is **truly
usable**: ACP must be enabled, dispatch must not be disabled, the current
session must not be sandbox-blocked, and a runtime backend must be
loaded. If those conditions are not met, ACP plugin skills and
`sessions_spawn` ACP guidance stay hidden so the agent does not suggest
an unavailable backend.

### First-run gotchas

- If `plugins.allow` is set, it is a restrictive plugin inventory and **must** include `acpx`; otherwise the installed ACP backend is intentionally blocked and `/acp doctor` reports the missing allowlist entry.
    - The Codex ACP adapter is staged with the `acpx` plugin and launched locally when possible.
    - Codex ACP runs with an isolated `CODEX_HOME`; OpenClaw copies only trusted project entries from the host Codex config and trusts the active workspace, leaving auth, notifications, and hooks on the host config.
    - Other target harness adapters may still be fetched on demand with `npx` the first time you use them.
    - Vendor auth still has to exist on the host for that harness.
    - If the host has no npm or network access, first-run adapter fetches fail until caches are pre-warmed or the adapter is installed another way.

  ### Runtime prerequisites

ACP launches a real external harness process. OpenClaw owns routing,
    background-task state, delivery, bindings, and policy; the harness
    owns its provider login, model catalog, filesystem behavior, and
    native tools.

    Before blaming OpenClaw, verify:

    - `/acp doctor` reports an enabled, healthy backend.
    - The target id is allowed by `acp.allowedAgents` when that allowlist is set.
    - The harness command can start on the Gateway host.
    - Provider auth is present for that harness (`claude`, `codex`, `gemini`, `opencode`, `droid`, etc.).
    - The selected model exists for that harness - model ids are not portable across harnesses.
    - The requested `cwd` exists and is accessible, or omit `cwd` and let the backend use its default.
    - Permission mode matches the work. Non-interactive sessions cannot click native permission prompts, so write/exec-heavy coding runs usually need an ACPX permission profile that can proceed headlessly.

  OpenClaw plugin tools and built-in OpenClaw tools are **not** exposed to
ACP harnesses by default. Enable the explicit MCP bridges in
[ACP agents - setup](/tools/acp-agents-setup) only when the harness
should call those tools directly.

## Supported harness targets

With the `acpx` backend, use these harness ids as `/acp spawn <id>`
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

**Spawn**

`/acp spawn claude --bind here`,
    `/acp spawn gemini --mode persistent --thread auto`, or explicit
    `/acp spawn codex --bind here`.
  
**Work**

Continue in the bound conversation or thread (or target the session
    key explicitly).
  
**Check state**

`/acp status`
  
**Tune**

`/acp model <provider/model>`,
    `/acp permissions <profile>`,
    `/acp timeout <seconds>`.
  
**Steer**

Without replacing context: `/acp steer tighten logging and continue`.
  
**Stop**

`/acp cancel` (current turn) or `/acp close` (session + bindings).
  
### Lifecycle details

- Spawn creates or resumes an ACP runtime session, records ACP metadata in the OpenClaw session store, and may create a background task when the run is parent-owned.
    - Parent-owned ACP sessions are treated as background work even when the runtime session is persistent; completion and cross-surface delivery go through the parent task notifier rather than acting like a normal user-facing chat session.
    - Task maintenance closes terminal or orphaned parent-owned one-shot ACP sessions. Persistent ACP sessions are preserved while an active conversation binding remains; stale persistent sessions without an active binding are closed so they cannot be silently resumed after the owning task is done or its task record is gone.
    - Bound follow-up messages go directly to the ACP session until the binding is closed, unfocused, reset, or expired.
    - Gateway commands stay local. `/acp ...`, `/status`, and `/unfocus` are never sent as normal prompt text to a bound ACP harness.
    - `cancel` aborts the active turn when the backend supports cancellation; it does not delete the binding or session metadata.
    - `close` ends the ACP session from OpenClaw's point of view and removes the binding. A harness may still keep its own upstream history if it supports resume.
    - The acpx plugin cleans up OpenClaw-owned wrapper and adapter process trees after `close`, and reaps stale OpenClaw-owned ACPX orphans during Gateway startup.
    - Idle runtime workers are eligible for cleanup after `acp.runtime.ttlMinutes`; stored session metadata remains available for `/acp sessions`.

  ### Native Codex routing rules

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
    arguments or rewrite Codex thread records. Use explicit ACP only
    when you want the ACP runtime/session model. The embedded Codex
    support boundary is documented in the
    [Codex harness v1 support contract](/plugins/codex-harness-runtime#v1-support-contract).

  ### Model / provider / runtime selection cheat sheet

- `openai-codex/*` - legacy Codex OAuth/subscription model route repaired by doctor.
    - `openai/*` - native Codex app-server embedded runtime for OpenAI agent turns.
    - `/codex ...` - native Codex conversation control.
    - `/acp ...` or `runtime: "acp"` - explicit ACP/acpx control.

  ### ACP-routing natural-language triggers

Triggers that should route to the ACP runtime:

    - "Run this as a one-shot Claude Code ACP session and summarize the result."
    - "Use Gemini CLI for this task in a thread, then keep follow-ups in that same thread."
    - "Run Codex through ACP in a background thread."

    OpenClaw picks `runtime: "acp"`, resolves the harness `agentId`,
    binds to the current conversation or thread when supported, and
    routes follow-ups to that session until close/expiry. Codex only
    follows this path when ACP/acpx is explicit or the native Codex
    plugin is unavailable for the requested operation.

    For `sessions_spawn`, `runtime: "acp"` is advertised only when ACP
    is enabled, the requester is not sandboxed, and an ACP runtime
    backend is loaded. `acp.dispatch.enabled=false` pauses automatic
    ACP thread dispatch but does not hide or block explicit
    `sessions_spawn({ runtime: "acp" })` calls. It targets ACP harness ids such as `codex`,
    `claude`, `droid`, `gemini`, or `opencode`. Do not pass a normal
    OpenClaw config agent id from `agents_list` unless that entry is
    explicitly configured with `agents.list[].runtime.type="acp"`;
    otherwise use the default sub-agent runtime. When an OpenClaw agent
    is configured with `runtime.type="acp"`, OpenClaw uses
    `runtime.acp.agent` as the underlying harness id.

  ## ACP versus sub-agents

Use ACP when you want an external harness runtime. Use **native Codex
app-server** for Codex conversation binding/control when the `codex`
plugin is enabled. Use **sub-agents** when you want OpenClaw-native
delegated runs.

| Area          | ACP session                           | Sub-agent run                      |
| ------------- | ---------------------------
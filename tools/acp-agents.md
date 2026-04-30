---
domain: tools
topic: "ACP Agents: Running Claude Code and External Coding Agents via ACP Protocol"
type: procedure
keywords:
  - ACP
  - Agent Communication Protocol
  - Claude Code
  - ACP agents
  - coding agents
  - ACP session
related:
  - tools/subagents
  - concepts/multi-agent
source: tools/acp-agents.md
---

ACP (Agent Communication Protocol) agents enable OpenClaw to run and communicate with external coding agents like Claude Code. ACP handles session binding, delivery, and sandboxing.

## ACP Overview

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/) sessions
let OpenClaw run external coding harnesses (for example Pi, Claude Code,
Cursor, Copilot, Droid, OpenClaw ACP, OpenCode, Gemini CLI, and other
supported ACPX harnesses) through an ACP backend plugin.

Each ACP session spawn is tracked as a [background task](/automation/tasks).

**ACP is the external-harness path, not the default Codex path.** The
native Codex app-server plugin owns `/codex ...` controls and the
`agentRuntime.id: "codex"` embedded runtime; ACP owns
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

Usually yes. Fresh installs ship the bundled `acpx` runtime plugin enabled
by default with a plugin-local pinned `acpx` binary that OpenClaw probes
and self-repairs on startup. Run `/acp doctor` for a readiness check.

OpenClaw only teaches agents about ACP spawning when ACP is **truly
usable**: ACP must be enabled, dispatch must not be disabled, the current
session must not be sandbox-blocked, and a runtime backend must be
loaded. If those conditions are not met, ACP plugin skills and
`sessions_spawn` ACP guidance stay hidden so the agent does not suggest
an unavailable backend.

    - If `plugins.allow` is set, it is a restrictive plugin inventory and **must** include `acpx`; otherwise the bundled default is intentionally blocked and `/acp doctor` reports the missing allowlist entry.
    - The bundled Codex ACP adapter is staged with the `acpx` plugin and launched locally when possible.
    - Other target harness adapters may still be fetched on demand with `npx` the first time you use them.
    - Vendor auth still has to exist on the host for that harness.
    - If the host has no npm or network access, first-run adapter fetches fail until caches are pre-warmed or the adapter is installed another way.

    ACP launches a real external harness process. OpenClaw owns routing,
    background-task state, delivery, bindings, and policy; the harness
    owns its provider login, model catalog, filesystem behavior, and
    native tools.

    Before blaming OpenClaw, verify:

    - `/acp doctor` reports an enabled, healthy backend.
    - The target id is allowed by `acp.allowedAgents` when that allowlist is set.
    - The harness command can start on the Gateway host.
    - Provider auth is present for that harness (`claude`, `codex`, `gemini`, `opencode`, `droid`, etc.).
    - The selected model exists for that harness — model ids are not portable across harnesses.
    - The requested `cwd` exists and is accessible, or omit `cwd` and let the backend use its default.
    - Permission mode matches the work. Non-interactive sessions cannot click native permission prompts, so write/exec-heavy coding runs usually need an ACPX p

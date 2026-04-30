---
domain: tools
topic: "Subagents: Spawning and Coordinating Child Agents for Parallel Work"
type: concept
keywords:
  - subagents
  - sub-agents
  - child agents
  - parallel agents
  - agent spawning
  - multi-agent tools
related:
  - concepts/multi-agent
  - tools/acp-agents
  - concepts/agent-loop
source: tools/subagents.md
---

Subagent tools allow the agent to spawn and coordinate child agents for parallel or specialized work.

Sub-agents are background agent runs spawned from an existing agent run.
They run in their own session (`agent:<agentId>:subagent:<uuid>`) and,
when finished, **announce** their result back to the requester chat
channel. Each sub-agent run is tracked as a
[background task](/automation/tasks).

Primary goals:

- Parallelize "research / long task / slow tool" work without blocking the main run.
- Keep sub-agents isolated by default (session separation + optional sandboxing).
- Keep the tool surface hard to misuse: sub-agents do **not** get session tools by default.
- Support configurable nesting depth for orchestrator patterns.

**Cost note:** each sub-agent has its own context and token usage by
default. For heavy or repetitive tasks, set a cheaper model for sub-agents
and keep your main agent on a higher-quality model. Configure via
`agents.defaults.subagents.model` or per-agent overrides. When a child
genuinely needs the requester's current transcript, the agent can request
`context: "fork"` on that one spawn.

## Slash command

Use `/subagents` to inspect or control sub-agent runs for the **current
session**:

```text
/subagents list
/subagents kill <id|#|all>
/subagents log <id|#> [limit] [tools]
/subagents info <id|#>
/subagents send <id|#> <message>
/subagents steer <id|#> <message>
/subagents spawn <agentId> <task> [--model <model>] [--thinking <level>]
```

`/subagents info` shows run metadata (status, timestamps, session id,
transcript path, cleanup). Use `sessions_history` for a bounded,
safety-filtered recall view; inspect the transcript path on disk when you
need the raw full transcript.

### Thread binding controls

These commands work on channels that support persistent thread bindings.
See [Thread supporting channels](#thread-supporting-channels) below.

```text
/focus <subagent-label|session-key|session-id|session-label>
/unfocus
/agents
/session idle <duration|off>
/session max-age <duration|off>
```

### Spawn behavior

`/subagents spawn` starts a background sub-agent as a user command (not an
internal relay) and sends one final completion update back to the
requester chat when the run finishes.

    - The spawn command is non-blocking; it returns a run id immediately.
    - On completion, the sub-agent announces a summary/result message back to the requester chat channel.
    - Completion is push-based. Once spawned, do **not** poll `/subagents list`, `sessions_list`, or `sessions_history` in a loop just to wait for it to finish; inspect status only on-demand for debugging or intervention.
    - On completion, OpenClaw best-effort closes tracked browser tabs/processes opened by that sub-agent session before the announce cleanup flow continues.

    - OpenClaw tries direct `agent` delivery first with a stable idempotency key.
    - If direct delivery fails, it falls back to queue routing.
    - If queue routing is still not available, the announce is retried with a short exponential backoff before final give-up.
    - Completion delivery keeps the resolved requester route: thread-bound or conversation-bound completion routes win when available; if the completion origin only provides a channel, OpenClaw fills the missing target/account from the requester session's resolved route (`lastChannel` / `lastTo` / `lastAccountId`) so direct delivery still works.

    The completion handoff to the requester session is runtime-generated
    internal context (not user-authored text) and includes:

    - `Result` — latest visible `assistant` reply text, otherwise sanitized latest tool/toolResult text. Terminal failed runs do not reuse captured reply text.
    - `Status` — `completed successfully` / `failed` / `timed out` / `unknown`.
    - Compact runtime/token stats.
    - A delivery instruction telling the requester agent to rewrite in normal assistant voice (not forward raw internal metadata).

    - `--model` and `--thinking` override defaults for that specific run.
    - Use `info`/`log` to inspect details and output after completion.
    - `/subagents spawn` is one-shot mode (`mode: "run"`). For persistent thread-bound sessions, use `sessions_spawn` with `thread: true` and `mode: "session"`.
    - For ACP harness sessions (Claude Code, Gemini CLI, OpenCode, or explicit Codex ACP/acpx), use `sessions_spawn` with `runtime: "acp"` when the tool advertises that runtime. See [ACP delivery model](/tools/acp-agents#delivery-model) when debugging completions or agent-to-agent loops. When the `codex` plugin is enabled, Codex chat/thread control should prefer `/codex ...` over ACP unless the user explicitly asks for ACP/acpx.
    - OpenClaw hides `runtime: "acp"` until ACP is enabled, the requester is not sandboxed, and a backend plugin such as `acpx` is loaded. `runtime: "acp"` expects an external ACP harness id, or an `agents.list[]` entry with `runtime.type="acp"`; use the default sub-agent runtime for normal OpenClaw config agents from `agents_list`.

## Context modes

Native sub-agents start isolated unless the caller explicitly asks to fork
the current transcript.

| Mode       | When to use it                                                                                                                         | Behavior                                                                          |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `isolated` | Fresh research, independent implementation, slow tool work, or anything that can be briefed in the task text                           | Creates a clean child transcript. This is the default and keeps token use lower.  |
| `fork`     | Work that depends on the current conversation, prior tool results, or nuanced instructions already present in the requester transcript | Branches the requester transcript into the child session before the child starts. |

Use `fork` sparingly. It is for context-sensitive delegation, not a
replacement for writing a clear task prompt.

## Tool: `sessions_spawn`

Starts a sub-agent run with `deliver: false` on the global `subagent` lane,
then runs an announce step and posts the announce reply to the requester
chat channel.

Availability depends on the caller's effective tool policy. The `coding` and
`full` profiles expose `sessions_spawn` by default. The `messaging` profile
does not; add `tools.alsoAllow: ["sessions_spawn", "sessions_yield",
"subagents"]` or use `tools.profile: "coding"` for agents that should delegate
work. Channel/group, provider, sandbox, and per-agent allow/deny policies can
still remove the tool after the profile stage. Use `/tools` from the same
session to confirm the effective tool list.

**Defaults:**

- **Model:** inherits the caller unless you set `agents.defaults.subagents.model` (or per-agent `agents.list[].subagents.model`); an explicit `sessions_spawn.model` still wins.
- **Thinking:** inherits the caller unless you set `agents.defaults.subagents.thinking` (or per-agent `agents.list[].subagents.thinking`); an explicit `sessions_spawn.thinking` still wins.
- **Run timeout:** if `sessions_spawn.runTimeoutSeconds` is omitted, OpenClaw uses `agents.defaults.subagents.runTimeoutSeconds` when set; otherwise it falls back to `0` (no timeout).

### Tool parameters

<ParamField path="task" type="string" required>
  The task description for the sub-agent.
</ParamField>
<ParamField path="label" type="string">
  Optional human-readable label.
</ParamField>
<ParamField path="agentId" type="string">
  Spawn under another agent id when allowed by `subagents.allowAgents`.
</ParamField>
<ParamField path="runtime" type='"subagent" | "acp"' default="subagent">
  `acp` is only for external ACP harnesses (`claude`, `droid`, `gemini`, `opencode`, or explicitly requested Codex ACP/acpx) and for `agents.list[]` entries whose `runtime.type` is `acp`.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  ACP-only. Resumes an existing ACP harness session when `runtime: "acp"`; ignored for native sub-agent spawns.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  ACP-only. Streams ACP run output to the parent session when `runtime: "acp"`; omit for native sub-agent spawns.
</ParamField>
<ParamField path="model" type="string">
  Override the sub-agent model. Invalid values are skipped and the sub-agent runs on the default model with a warning in the tool result.
</ParamField>
<ParamField path="thinking" type="string">
  Override thinking level for the sub-agent run.
</ParamField>
<ParamField path="runTimeoutSeconds" type="number">
  Defaults to `agents.defaults.subagents.runTimeoutSeconds` when set, otherwise `0`. When set, the sub-agent run is aborted after N seconds.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  When `true`, requests channel thread binding for this sub-agent session.
</ParamField>
<Par

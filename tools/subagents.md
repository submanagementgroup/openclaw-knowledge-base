---
domain: tools
topic: "Subagents: Spawning and Coordinating Sub-Agents"
type: concept
keywords:
  - subagents
  - sub-agents
  - agent spawning
  - agent-send
  - parallel agents
  - task delegation
source: tools/subagents.md
---

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

> **Note:** **Cost note:** each sub-agent has its own context and token usage by
default. For heavy or repetitive tasks, set a cheaper model for sub-agents
and keep your main agent on a higher-quality model. Configure via
`agents.defaults.subagents.model` or per-agent overrides. When a child
    genuinely needs the requester's current transcript, the agent can request
    `context: "fork"` on that one spawn. Thread-bound subagent sessions default
    to `context: "fork"` because they branch the current conversation into a
    follow-up thread.


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

Use top-level [`/steer <message>`](/tools/steer) to steer the current requester session's active run. Use `/subagents steer <id|#> <message>` when the target is a child run.

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

### Non-blocking, push-based completion

- The spawn command is non-blocking; it returns a run id immediately.
    - On completion, the sub-agent announces a summary/result message back to the requester chat channel.
    - Agent turns that need child results should call `sessions_yield` after spawning required work. That ends the current turn and lets completion events arrive as the next model-visible message.
    - Completion is push-based. Once spawned, do **not** poll `/subagents list`, `sessions_list`, or `sessions_history` in a loop just to wait for it to finish; inspect status only on-demand for debugging or intervention.
    - Child output is a report/evidence for the requester agent to synthesize. It is not user-authored instruction text and cannot override system, developer, or user policy.
    - On completion, OpenClaw best-effort closes tracked browser tabs/processes opened by that sub-agent session before the announce cleanup flow continues.

  ### Manual-spawn delivery resilience

- OpenClaw hands completions back to the requester session through an `agent` turn with a stable idempotency key.
    - If the requester run is still active, OpenClaw first tries to wake/steer that run instead of starting a second visible reply path.
    - If the requester-agent completion handoff fails or produces no visible output, OpenClaw treats delivery as failed and falls back to queue routing/retry. It does not raw-send the child result directly to the external chat.
    - If direct handoff cannot be used, it falls back to queue routing.
    - If queue routing is still not available, the announce is retried with a short exponential backoff before final give-up.
    - Completion delivery keeps the resolved requester route: thread-bound or conversation-bound completion routes win when available; if the completion origin only provides a channel, OpenClaw fills the missing target/account from the requester session's resolved route (`lastChannel` / `lastTo` / `lastAccountId`) so direct delivery still works.

  ### Completion handoff metadata

The completion handoff to the requester session is runtime-generated
    internal context (not user-authored text) and includes:

    - `Result` — latest visible `assistant` reply text, otherwise sanitized latest tool/toolResult text. Terminal failed runs do not reuse captured reply text.
    - `Status` — `completed successfully` / `failed` / `timed out` / `unknown`.
    - Compact runtime/token stats.
    - A delivery instruction telling the requester agent to rewrite in normal assistant voice (not forward raw internal metadata).

  ### Modes and ACP runtime

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

### Delegation prompt mode

`agents.defaults.subagents.delegationMode` controls prompt guidance only; it does not change tool policy or enforce delegation.

- `suggest` (default): keep the standard prompt nudge to use sub-agents for larger or slower work.
- `prefer`: tell the main agent to stay responsive and delegate anything more involved than a direct reply through `sessions_spawn`.

Per-agent overrides use `agents.list[].subagents.delegationMode`.

```json5
{
  agents: {
    defaults: {
      subagents: {
        delegationMode: "prefer",
        maxConcurrent: 4,
      },
    },
    list: [
      {
        id: "coordinator",
        subagents: { delegationMode: "prefer" },
      },
    ],
  },
}
```

### Tool parameters


  The task description for the sub-agent.


  Optional stable handle for later `subagents` targeting. Must match `[a-z][a-z0-9_]{0,63}` and cannot be reserved targets such as `last` or `all`. Prefer it when the coordinator may need to steer, kill, or identify a specific child after spawning several children.


  Optional human-readable label.


  Spawn under another agent id when allowed by `subagents.allowAgents`.


  `acp` is only for external ACP harnesses (`claude`, `droid`, `gemini`, `opencode`, or explicitly requested Codex ACP/acpx) and for `agents.list[]` entries whose `runtime.type` is `acp`.


  ACP-only. Resumes an existing ACP harness session when `runtime: "acp"`; ignored for native sub-agent spawns.


  ACP-only. Streams ACP run output to the parent session when `runtime: "acp"`; omit for native sub-agent spawns.


  Override the sub-agent model. Invalid values are skipped and the sub-agent runs on the default model with a warning in the tool result.


  Override thinking level for the sub-agent run.


  Defaults to `agents.defaults.subagents.runTimeoutSeconds` when set, otherwise `0`. When set, the sub-agent run is aborted after N seconds.


  When `true`, requests channel thread binding for this sub-agent session.


  If `thread: true` and `mode` omitted, default becomes `session`. `mode: "session"` requires `thread: true`.


  `"delete"` archives immediately after announce (still keeps the transcript via rename).


  `require` rejects spawn unless the target child runtime is sandboxed.


  `fork` branches the requester's current transcript into the child session. Native sub-agents only. Thread-bound spawns default to `fork`; non-thread spawns default to `isolated`.


> **Note:** `sessions_spawn` does **not** accept channel-delivery params (`target`,
`channel`, `to`, `threadId`, `replyTo`, `transport`). For delivery, use
`message`/`sessions_send` from the spawned run.


### Task names and targeting

`taskName` is a model-facing handle for orchestration, not a session key.
Use it for stable child names such as `review_subagents`,
`linux_validation`, or `docs_update` when a coordinator may need to steer
or kill that child later.

Target resolution accepts exact `taskName` matches and unambiguous
prefixes. Matching is scoped to the same active/recent target window used
by numbered `/subagents` targets, so a stale completed child does not make
a reused handle ambiguous. If two active or recent children share the same
`taskName`, the target is ambiguous; use the list index, session key, or
run id instead.

The reserved targets `last` and `all` are not valid `taskName` values
because they already have control meanings.

## Tool: `sessions_yield`

Ends the current model turn and waits for runtime events, primarily
sub-agent completion events, to arrive as the next message. Use it after
spawning required child work when the requester cannot produce a final
answer until those completions arrive.

`sessions_yield` is the waiting primitive. Do not replace it with polling
loops over `subagents`, `sessions_list`, `sessions_history`, shell
`sleep`, or process polling just to detect child completion.

Only use `sessions_yield` when the session's effective tool list includes
it. Some minimal or custom tool profiles may expose `sessions_spawn` and
`subagents` without exposing `sessions_yield`; in that case, do not invent
a polling loop just to wait for completion.

When active children exist, OpenClaw injects a compact runtime-generated
`Active Subagents` prompt block into normal turns so the requester can see
the current child sessions, run ids, statuses, labels, tasks, and
`taskName` aliases without polling. The task and label fields in that
block are quoted as data, not instructions, because they can originate
from user/model-provided spawn arguments.

## Tool: `subagents`

Lists, steers, or kills spawned sub-agent runs owned by the requester
session. It is scoped to the current requester; a child can only
see/control its own controlled children.

Use `subagents` for on-demand status, debugging, steering, or killing.
Use `sessions_yield` to wait for completion events.

## Thread-bound sessions

When thread bindings are enabled for a channel, a sub-agent can stay bound
to a thread so follow-up user messages in that thread keep routing to the
same sub-agent session.

### Thread supporting channels

**Discord** is currently the only supported channel. It supports
persistent thread-bound subagent sessions (`sessions_spawn` with
`thread: true`), manual thread controls (`/focus`, `/unfocus`, `/agents`,
`/session idle`, `/session max-age`), and adapter key
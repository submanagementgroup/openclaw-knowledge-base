---
domain: tools
topic: "ACP Agents: Advanced Patterns and Reference"
type: reference
keywords:
  - ACP
  - ACP advanced
  - ACP config
  - ACP protocol reference
source: tools/acp-agents.md
---

---------- | ---------------------------------- |
| Runtime       | ACP backend plugin (for example acpx) | OpenClaw native sub-agent runtime  |
| Session key   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Main commands | `/acp ...`                            | `/subagents ...`                   |
| Spawn tool    | `sessions_spawn` with `runtime:"acp"` | `sessions_spawn` (default runtime) |

See also [Sub-agents](/tools/subagents).

## How ACP runs Claude Code

For Claude Code through ACP, the stack is:

1. OpenClaw ACP session control plane.
2. Official `@openclaw/acpx` runtime plugin.
3. Claude ACP adapter.
4. Claude-side runtime/session machinery.

ACP Claude is a **harness session** with ACP controls, session resume,
background-task tracking, and optional conversation/thread binding.

CLI backends are separate text-only local fallback runtimes - see
[CLI Backends](/gateway/cli-backends).

For operators, the practical rule is:

- **Want `/acp spawn`, bindable sessions, runtime controls, or persistent harness work?** Use ACP.
- **Want simple local text fallback through the raw CLI?** Use CLI backends.

## Bound sessions

### Mental model

- **Chat surface** - where people keep talking (Discord channel, Telegram topic, iMessage chat).
- **ACP session** - the durable Codex/Claude/Gemini runtime state OpenClaw routes to.
- **Child thread/topic** - an optional extra messaging surface created only by `--thread ...`.
- **Runtime workspace** - the filesystem location (`cwd`, repo checkout, backend workspace) where the harness runs. Independent of the chat surface.

### Current-conversation binds

`/acp spawn <harness> --bind here` pins the current conversation to the
spawned ACP session - no child thread, same chat surface. OpenClaw keeps
owning transport, auth, safety, and delivery. Follow-up messages in that
conversation route to the same session; `/new` and `/reset` reset the
session in place; `/acp close` removes the binding.

Examples:

```text
/codex bind                                              # native Codex bind, route future messages here
/codex model gpt-5.4                                     # tune the bound native Codex thread
/codex stop                                              # control the active native Codex turn
/acp spawn codex --bind here                             # explicit ACP fallback for Codex
/acp spawn codex --thread auto                           # may create a child thread/topic and bind there
/acp spawn codex --bind here --cwd /workspace/repo       # same chat binding, Codex runs in /workspace/repo
```

### Binding rules and exclusivity

- `--bind here` and `--thread ...` are mutually exclusive.
    - `--bind here` only works on channels that advertise current-conversation binding; OpenClaw returns a clear unsupported message otherwise. Bindings persist across gateway restarts.
    - On Discord, `spawnSessions` gates child thread creation for `--thread auto|here` - not `--bind here`.
    - If you spawn to a different ACP agent without `--cwd`, OpenClaw inherits the **target agent's** workspace by default. Missing inherited paths (`ENOENT`/`ENOTDIR`) fall back to the backend default; other access errors (e.g. `EACCES`) surface as spawn errors.
    - Gateway management commands stay local in bound conversations - `/acp ...` commands are handled by OpenClaw even when normal follow-up text routes to the bound ACP session; `/status` and `/unfocus` also stay local whenever command handling is enabled for that surface.

  ### Thread-bound sessions

When thread bindings are enabled for a channel adapter:

    - OpenClaw binds a thread to a target ACP session.
    - Follow-up messages in that thread route to the bound ACP session.
    - ACP output is delivered back to the same thread.
    - Unfocus/close/archive/idle-timeout or max-age expiry removes the binding.
    - `/acp close`, `/acp cancel`, `/acp status`, `/status`, and `/unfocus` are Gateway commands, not prompts to the ACP harness.

    Required feature flags for thread-bound ACP:

    - `acp.enabled=true`
    - `acp.dispatch.enabled` is on by default (set `false` to pause automatic ACP thread dispatch; explicit `sessions_spawn({ runtime: "acp" })` calls still work).
    - Channel-adapter thread session spawns enabled (default: `true`):
      - Discord: `channels.discord.threadBindings.spawnSessions=true`
      - Telegram: `channels.telegram.threadBindings.spawnSessions=true`

    Thread binding support is adapter-specific. If the active channel
    adapter does not support thread bindings, OpenClaw returns a clear
    unsupported/unavailable message.

  ### Thread-supporting channels

- Any channel adapter that exposes session/thread binding capability.
    - Current built-in support: **Discord** threads/channels, **Telegram** topics (forum topics in groups/supergroups and DM topics).
    - Plugin channels can add support through the same binding interface.

  ## Persistent channel bindings

For non-ephemeral workflows, configure persistent ACP bindings in
top-level `bindings[]` entries.

### Binding model


  Marks a persistent ACP conversation binding.


  Identifies the target conversation. Per-channel shapes:

- **Discord channel/thread:** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Slack channel/DM:** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`. Prefer stable Slack ids; channel bindings also match replies inside that channel's threads.
- **Telegram forum topic:** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **iMessage DM/group:** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`. Prefer `chat_id:*` for stable group bindings.



  The owning OpenClaw agent id.


  Optional ACP override.


  Optional operator-facing label.


  Optional runtime working directory.


  Optional backend override.


### Runtime defaults per agent

Use `agents.list[].runtime` to define ACP defaults once per agent:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (harness id, e.g. `codex` or `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

**Override precedence for ACP bound sessions:**

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. Global ACP defaults (e.g. `acp.backend`)

### Example

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### Behavior

- OpenClaw ensures the configured ACP session exists before use.
- Messages in that channel or topic route to the configured ACP session.
- In bound conversations, `/new` and `/reset` reset the same ACP session key in place.
- Temporary runtime bindings (for example created by thread-focus flows) still apply where present.
- For cross-agent ACP spawns without an explicit `cwd`, OpenClaw inherits the target agent workspace from agent config.
- Missing inherited workspace paths fall back to the backend default cwd; non-missing access failures surface as spawn errors.

## Start ACP sessions

Two ways to start an ACP session:

**From sessions_spawn:**

Use `runtime: "acp"` to start an ACP session from an agent turn or
    tool call.

    ```json
    {
      "task": "Open the repo and summarize failing tests",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    > **Note:** `runtime` defaults to `subagent`, so set `runtime: "acp"` explicitly
    for ACP sessions. If `agentId` is omitted, OpenClaw uses
    `acp.defaultAgent` when configured. `mode: "session"` requires
    `thread: true` to keep a persistent bound conversation.
    

  
**From /acp command:**

Use `/acp spawn` for explicit operator control from chat.

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    Key flags:

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    See [Slash commands](/tools/slash-commands).

  
### `sessions_spawn` parameters


  Initial prompt sent to the ACP session.


  Must be `"acp"` for ACP sessions.


  ACP target harness id. Falls back to `acp.defaultAgent` if set.


  Request thread binding flow where supported.


  `"run"` is one-shot; `"session"` is persistent. If `thread: true` and
  `mode` is omitted, OpenClaw may default to persistent behaviour per
  runtime path. `mode: "session"` requires `thread: true`.


  Requested runtime working directory (validated by backend/runtime
  policy). If omitted, ACP spawn inherits the target agent workspace
  when configured; missing inherited paths fall back to backend
  defaults, while real access errors are returned.


  Operator-facing label used in session/banner text.


  Resume an existing ACP session instead of creating a new one. The
  agent replays its conversation history via `session/load`. Requires
  `runtime: "acp"`.


  `"parent"` streams initial ACP run progress summaries back to the
  requester session as system events. Accepted responses include
  `streamLogPath` pointing to a session-scoped JSONL log
  (`<sessionId>.acp-stream.jsonl`) you can tail for full relay history.


  Aborts the ACP child turn after N seconds. `0` keeps the turn on the
  gateway's no-timeout path. The same value is applied to the Gateway
  run and ACP runtime so stalled/quota-exhausted harnesses do not
  occupy the parent agent lane indefinitely.


  Explicit model override for the ACP child session. Codex ACP spawns
  normalize OpenClaw Codex refs such as `openai-codex/gpt-5.4` to Codex
  ACP startup config before `session/new`; slash forms such as
  `openai-codex/gpt-5.4/high` also set Codex ACP reasoning effort.
  Other harnesses must advertise ACP `models` and support
  `session/set_model`; otherwise OpenClaw/acpx fails clearly instead of
  silently falling back to the target agent default.


  Explicit thinking/reasoning effort. For Codex ACP, `minimal` maps to
  low effort, `low`/`medium`/`high`/`xhigh` map directly, and `off`
  omits the reasoning-effort startup override.


## Spawn bind and thread modes

**--bind here|off:**

| Mode   | Behavior                                                               |
    | ------ | ---------------------------------------------------------------------- |
    | `here` | Bind the current active conversation in place; fail if none is active. |
    | `off`  | Do not create a current-conversation binding.                          |

    Notes:

    - `--bind here` is the simplest operator path for "make this channel or chat Codex-backed."
    - `--bind here` does not create a child thread.
    - `--bind here` is only available on channels that expose current-conversation binding support.
    - `--bind` and `--thread` cannot be combined in the same `/acp spawn` call.

  
**--thread auto|here|off:**

| Mode   | Behavior                                                                                            |
    | ------ | --------------------------------------------------------------------------------------------------- |
    | `auto` | In an active thread: bind that thread. Outside a thread: create/bind a child thread when supported. |
    | `here` | Require current active thread; fail if not in one.                                                  |
    | `off`  | No binding. Session starts unbound.                                                                 |

    Notes:

    - On non-thread binding surfaces, default behavior is effectively `off`.
    - Thread-bound spawn requires channel policy support:
      - Discord: `channels.discord.threadBindings.spawnSessions=true`
      - Telegram: `channels.telegram.threadBindings.spawnSessions=true`
    - Use `--bind here` when you want to pin the current conversation without creating a child thread.

  
## Delivery model

ACP sessions can be either interactive workspaces or parent-owned
background work. The delivery path depends on that shape.

### Interactive ACP sessions

Interactive sessions are meant to keep talking on a visible chat
    surface:

    - `/acp spawn ... --bind here` binds the current conversation to the ACP session.
    - `/acp spawn ... --thread ...` binds a channel thread/topic to the ACP ses
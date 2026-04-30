---
domain: concepts
topic: "Session Model: Persistent Conversations, Session Keys, Session Tool, and Pruning"
type: concept
keywords:
  - session
  - session key
  - session persistence
  - session pruning
  - sessionKey
  - conversation history
related:
  - concepts/compaction
  - concepts/agent-loop
  - concepts/context
source:
  - concepts/session.md
  - concepts/session-tool.md
  - concepts/session-pruning.md
---

OpenClaw sessions are the persistent conversation histories for each agent-user combination. Sessions are stored on disk and persist across gateway restarts.

## Session Model

OpenClaw organizes conversations into **sessions**. Each message is routed to a
session based on where it came from -- DMs, group chats, cron jobs, etc.

## How messages are routed

| Source          | Behavior                  |
| --------------- | ------------------------- |
| Direct messages | Shared session by default |
| Group chats     | Isolated per group        |
| Rooms/channels  | Isolated per room         |
| Cron jobs       | Fresh session per run     |
| Webhooks        | Isolated per hook         |

## DM isolation

By default, all DMs share one session for continuity. This is fine for
single-user setups.

If multiple people can message your agent, enable DM isolation. Without it, all
users share the same conversation context -- Alice's private messages would be
visible to Bob.

**The fix:**

```json5
{
  session: {
    dmScope: "per-channel-peer", // isolate by channel + sender
  },
}
```

Other options:

- `main` (default) -- all DMs share one session.
- `per-peer` -- isolate by sender (across channels).
- `per-channel-peer` -- isolate by channel + sender (recommended).
- `per-account-channel-peer` -- isolate by account + channel + sender.

If the same person contacts you from multiple channels, use
`session.identityLinks` to link their identities so they share one session.

### Dock linked channels

Dock commands let a user move the current direct-chat session's reply route to
another linked channel without starting a new session. See
[Channel docking](/concepts/channel-docking) for examples, config, and
troubleshooting.

Verify your setup with `openclaw security audit`.

## Session lifecycle

Sessions are reused until they expire:

- **Daily reset** (default) -- new session at 4:00 AM local time on the gateway
  host. Daily freshness is based on when the current `sessionId` started, not
  on later metadata writes.
- **Idle reset** (optional) -- new session after a period of inactivity. Set
  `session.reset.idleMinutes`. Idle freshness is based on the last real
  user/channel interaction, so heartbeat, cron, and exec system events do not
  keep the session alive.
- **Manual reset** -- type `/new` or `/reset` in chat. `/new <model>` also
  switches the model.

When both daily and idle resets are configured, whichever expires first wins.
Heartbeat, cron, exec, and other system-event turns may write session metadata,
but those writes do not extend daily or idle reset freshness. When a reset
rolls the session, queued system-event notices for the old session are
discarded so stale background updates are not prepended to the first prompt in
the new session.

Sessions with an active provider-owned CLI session are not cut by the implicit
daily default. Use `/reset` or configure `session.reset` explicitly when those
sessions should expire on a timer.

## Where state lives

All session state is owned by the **gateway**. UI clients query the gateway for
session data.

- **Store:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- **Transcripts:** `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

`sessions.json` keeps separate lifecycle timestamps:

- `sessionStartedAt`: when the current `sessionId` began; daily reset uses this.
- `lastInteractionAt`: last user/channel interaction that extends idle lifetime.
- `updatedAt`: last store-row mutation; useful for listing and pruning, but not
  authoritative for daily/idle reset freshness.

Older rows without `sessionStartedAt` are resolved from the transcript JSONL
session header when available. If an older row also lacks `lastInteractionAt`,
idle freshness falls back to that session start time, not to later bookkeeping
writes.

## Session maintenance

OpenClaw automatically bounds session storage over time. By default, it runs
in `warn` mode (reports what would be cleaned). Set `session.maintenance.mode`
to `"enforce"` for automatic cleanup:

```json5
{
  session: {
    maintenance: {
      mode: "enforce",
      pruneAfter: "30d",
      maxEntries: 500,
    },

## Session Tool

The session tool allows the agent to manage its own session:

OpenClaw gives agents tools to work across sessions, inspect status, and
orchestrate sub-agents.

## Available tools

| Tool               | What it does                                                                |
| ------------------ | --------------------------------------------------------------------------- |
| `sessions_list`    | List sessions with optional filters (kind, label, agent, recency, preview)  |
| `sessions_history` | Read the transcript of a specific session                                   |
| `sessions_send`    | Send a message to another session and optionally wait                       |
| `sessions_spawn`   | Spawn an isolated sub-agent session for background work                     |
| `sessions_yield`   | End the current turn and wait for follow-up sub-agent results               |
| `subagents`        | List, steer, or kill spawned sub-agents for this session                    |
| `session_status`   | Show a `/status`-style card and optionally set a per-session model override |

These tools are still subject to the active tool profile and allow/deny
policy. `tools.profile: "coding"` includes the full session orchestration
set, including `sessions_spawn`, `sessions_yield`, and `subagents`.
`tools.profile: "messaging"` includes cross-session messaging tools
(`sessions_list`, `sessions_history`, `sessions_send`, `session_status`) but
does not include sub-agent spawning. To keep a messaging profile and still
allow native delegation, add:

```json5
{
  tools: {
    profile: "messaging",
    alsoAllow: ["sessions_spawn", "sessions_yield", "subagents"],
  },
}
```

Group, provider, sandbox, and per-agent policies can still remove those tools
after the profile stage. Use `/tools` from the affected session to inspect the
effective tool list.

## Listing and reading sessions

`sessions_list` returns sessions with their key, agentId, kind, channel, model,
token counts, and timestamps. Filter by kind (`main`, `group`, `cron`, `hook`,
`node`), exact `label`, exact `agentId`, search text, or recency
(`activeMinutes`). When you need mailbox-style triage, it can also ask for a
visibility-scoped derived title, a last-message preview snippet, or bounded
recent messages on each row. Derived titles and previews are produced only for
sessions the caller can already see under the configured session tool
visibility policy, so unrelated sessions stay hidden.

`sessions_history` fetches the conversation transcript for a specific session.
By default, tool results are excluded -- pass `includeTools: true` to see them.
The returned view is intentionally bounded and safety-filtered:

- assistant text is normalized before recall:
  - thinking tags are stripped
  - `<relevant-memories>` / `<relevant_memories>` scaffolding blocks are stripped
  - plain-text tool-call XML payload blocks such as `<tool_call>...</tool_call>`,
    `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, and
    `<function_calls>...</function_calls>` ar

## Session Pruning

Session pruning trims **old tool results** from the context before each LLM
call. It reduces context bloat from accumulated tool outputs (exec results, file
reads, search results) without rewriting normal conversation text.

Pruning is in-memory only -- it does not modify the on-disk session transcript.
Your full history is always preserved.

## Why it matters

Long sessions accumulate tool output that inflates the context window. This
increases cost and can force [compaction](/concepts/compaction) sooner than
necessary.

Pruning is especially valuable for **Anthropic prompt caching**. After the cache
TTL expires, the next request re-caches the full prompt. Pruning reduces the
cache-write size, directly lowering cost.

## How it works

1. Wait for the cache TTL to expire (default 5 minutes).
2. Find old tool results for normal pruning (conversation text is left alone).
3. **Soft-trim** oversized results -- keep the head and tail, insert `...`.
4. **Hard-clear** the rest -- replace with a placeholder.
5. Reset the TTL so follow-up requests reuse the fresh cache.

## Legacy image cleanup

OpenClaw also builds a separate idempotent replay view for sessions that
persist raw image blocks or prompt-hydration media markers in history.

- It preserves the **3 most recent completed turns** byte-for-byte so prompt
  cache prefixes for recent follow-ups stay stable.
- In the replay view, older already-processed image blocks from `user` or
  `toolResult` history can be replaced with
  `[image data removed - already processed by model]`.
- Older textual media references such as `[media attached: ...]`,
  `[Image: source: ...]`, and `media://inbound/...` can be replaced with
  `[media reference removed - already processed by model]`. Current-turn
  attachment markers stay intact so vision models can still hydrate fresh
  images.
- The raw session transcript is not rewritten, so history viewers can still
  render the original message entries and their images.
- This is separate fr

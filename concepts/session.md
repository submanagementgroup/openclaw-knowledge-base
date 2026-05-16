---
domain: concepts
topic: "Sessions: Conversation Persistence and Management"
type: concept
keywords:
  - sessions
  - session
  - conversation session
  - session pruning
  - session tool
  - session management
source: ['concepts/session.md', 'concepts/session-pruning.md', 'concepts/session-tool.md']
---

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

> **Note:** If multiple people can message your agent, enable DM isolation. Without it, all
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

> **Note:** If the same person contacts you from multiple channels, use
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
  },
}
```

For production-sized `maxEntries` limits, Gateway runtime writes use a small high-water buffer and clean back down to the configured cap in batches. Session store reads do not prune or cap entries during Gateway startup. This avoids running full store cleanup on every startup or isolated cron session. `openclaw sessions cleanup --enforce` applies the cap immediately.

Maintenance preserves durable external conversation pointers, including group
sessions and thread-scoped chat sessions, while still allowing synthetic cron,
hook, heartbeat, ACP, and sub-agent entries to age out.

If you previously used direct-message isolation and later returned
`session.dmScope` to `main`, preview stale peer-keyed DM rows with
`openclaw sessions cleanup --dry-run --fix-dm-scope`. Applying the same flag
retires those old direct-DM rows and keeps their transcripts as deleted
archives.

Preview with `openclaw sessions cleanup --dry-run`.

## Inspecting sessions

- `openclaw status` -- session store path and recent activity.
- `openclaw sessions --json` -- all sessions (filter with `--active <minutes>`).
- `/status` in chat -- context usage, model, and toggles.
- `/context list` -- what is in the system prompt.

## Further reading

- [Session Pruning](/concepts/session-pruning) -- trimming tool results
- [Compaction](/concepts/compaction) -- summarizing long conversations
- [Session Tools](/concepts/session-tool) -- agent tools for cross-session work
- [Session Management Deep Dive](/reference/session-management-compaction) --
  store schema, transcripts, send policy, origin metadata, and advanced config
- [Multi-Agent](/concepts/multi-agent) — routing and session isolation across agents
- [Background Tasks](/automation/tasks) — how detached work creates task records with session references
- [Channel Routing](/channels/channel-routing) — how inbound messages are routed to sessions

## Related

- [Session pruning](/concepts/session-pruning)
- [Session tools](/concepts/session-tool)
- [Command queue](/concepts/queue)


## Session Pruning

Session pruning trims **old tool results** from the context before each LLM
call. It reduces context bloat from accumulated tool outputs (exec results, file
reads, search results) without rewriting normal conversation text.

> **Note:** Pruning is in-memory only -- it does not modify the on-disk session transcript.
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
- This is separate from normal cache-TTL pruning. It exists to stop repeated
  image payloads or stale media refs from busting prompt caches on later turns.

## Smart defaults

OpenClaw auto-enables pruning for Anthropic profiles:

| Profile type                                            | Pruning enabled | Heartbeat |
| ------------------------------------------------------- | --------------- | --------- |
| Anthropic OAuth/token auth (including Claude CLI reuse) | Yes             | 1 hour    |
| API key                                                 | Yes             | 30 min    |

If you set explicit values, OpenClaw does not override them.

## Enable or disable

Pruning is off by default for non-Anthropic providers. To enable:

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

To disable: set `mode: "off"`.

## Pruning vs compaction

|            | Pruning            | Compaction              |
| ---------- | ------------------ | ----------------------- |
| **What**   | Trims tool results | Summarizes conversation |
| **Saved?** | No (per-request)   | Yes (in transcript)     |
| **Scope**  | Tool results only  | Entire conversation     |

They complement each other -- pruning keeps tool output lean between
compaction cycles.

## Further reading

- [Compaction](/concepts/compaction) -- summarization-based context reduction
- [Gateway Configuration](/gateway/configuration) -- all pruning config knobs
  (`contextPruning.*`)

## Related

- [Session management](/concepts/session)
- [Session tools](/concepts/session-tool)
- [Context engine](/concepts/context-engine)


## Session Tool

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
    `<function_calls>...</function_calls>` are stripped, including truncated
    payloads that never close cleanly
  - downgraded tool-call/result scaffolding such as `[Tool Call: ...]`,
    `[Tool Result ...]`, and `[Historical context ...]` is stripped
  - leaked model control tokens such as `<|assistant|>`, other ASCII
    `<|...|>` tokens, and full-width `<｜...｜>` variants are stripped
  - malformed MiniMax tool-call XML such as `<invoke ...>` /
    `</minimax:tool_call>` is stripped
- credential/token-like text is redacted before it is returned
- long text blocks are truncated
- very large histories can drop older rows or replace an oversized row with
  `[sessions_history omitted: message too large]`
- the tool reports summary flags such as `truncated`, `droppedMessages`,
  `contentTruncated`, `contentRedacted`, and `bytes`

Both tools accept either a **session key** (like `"main"`) or a **session ID**
from a previous list call.

If you need the exact byte-for-byte transcript, inspect the transcript file on
disk instead of treating `sessions_history` as a raw dump.

## Sending cross-session messages

`sessions_send` delivers a message to another session and optionally waits for
the response:

- **Fire-and-forget:** set `timeoutSeconds: 0` to enqueue and return
  immediately.
- **Wait for reply:** set a timeout and get the response inline.

Thread-scoped chat sessions, such as Slack or Discord keys ending in
`:thread:<id>`, are not valid `sessions_send` targets. Use the parent channel
session key for inter-agent coordination so tool-routed messages do not appear
inside an active human-facing thread.

Messages and A2A follow-up replies are marked as inter-session data in the
receiving prompt (`[Inter-session message ... isUser=false]`) and in transcript
provenance. The receiving agent should treat them as tool-routed data, not as a
direct end-user-authored instruction.

After the target responds, OpenClaw can run a **reply-back loop** where the
agents alternate messages (up to `session.agentToAgent.maxPingPongTurns`, range
0-20, default 5). The target agent can reply
`REPLY_SKIP` to stop early.

## Status and orchestration helpers

`session_status` is the lightweight `/status`-equivalent tool for the current
or another visible session. It reports usage, time, model/runtime state, and
linked background-task context when present. Like `/status`, it can backfill
sparse token/cache counters from the latest transcript usage entry, and
`model=default` clears a per-session override. Use `sessionKey="current"` for
the caller's current session; visible client labels such as `openclaw-tui` are
not session keys.

`sessions_yield` intentionally ends the current turn so the next message can be
the follow-up event you are waiting for. Use it after spawning sub-agents when
you want completion results to arrive as the next message instead of building
poll loops.

`subagents` is the control-plane helper for already spawned OpenClaw
sub-agents. It supports:

- `action: "list"` to inspect active/recent runs
- `action: "steer"` to send follow-up guidance to a running child
- `action: "kill"` to stop one child or `all`

## Spawning sub-agents

`sessions_spawn` creates an isolated session for a background task by default.
It is always non-blocking -- it returns immediately with a `runId` and
`childSessionKey`.

Key options:

- `runtime: "subagent"` (default) or `"acp"` for external harness agents.
- `model` and `thinking` overrides for the child session.
- `thread: true` to bind the spawn to a chat thread (Discord, Slack, etc.).
- `sandbox: "require"` to enforce sandboxing on the child.
- `context: "fork"` for native sub-agents when the child needs the current
  requester transcript; omit it or use `context: "isolated"` for a clean child.
  Thread-bound native sub-agents default to `context: "fork"` unless
  `threadBindings.defaultSpawnContext` says otherwise.

Default leaf sub-agents do not get session tools. When
`maxSpawnDepth >= 2`, depth-1 orchestrator sub-agents additionally receive
`sessions_spawn`, `subagents`, `sessions_list`, and `sessions_history` so they
can manage their own children. Leaf runs still do not get recursive
orchestration tools.

After completion, an announce step posts the result to the requester's channel.
Completion delivery preserves bound thread/topic routing when available, and if
the completion origin only identifies a channel OpenClaw can still reuse the
requester session's stored route (`lastChannel` / `lastTo`) for direct
delivery.

For ACP-specific behavior, see [ACP Agents](/tools/acp-agents).

## Visibility

Session tools are scoped to limit what the agent can see:

| Level   | Scope                                    |
| ------- | ---------------------------------------- |
| `self`  | Only the current session                 |
| `tree`  | Current session + spawned sub-agents     |
| `agent` | All sessions for this agent              |
| `all`   | All sessions (cross-agent if configured) |

Default is `tree`. Sandboxed sessions are clamped to `tree` regardless of
config.

## Further reading

- [Session Management](/concepts/session) -- routing, lifecycle, maintenance
- [ACP Agents](/tools/acp-agents) -- external harness spawning
- [Multi-agent](/concepts/multi-agent) -- multi-agent architecture
- [Gateway Configuration](/gateway/configuration) -- session tool config knobs

## Related

- [Session management](/concepts/session)
- [Session pruning](/concepts/session-pruning)

---
domain: concepts
topic: "Message Queue and Queue Steering"
type: concept
keywords:
  - queue
  - message queue
  - queue steering
  - queue config
  - message ordering
  - concurrent requests
source: ['concepts/queue.md', 'concepts/queue-steering.md']
---

We serialize inbound auto-reply runs (all channels) through a tiny in-process queue to prevent multiple agent runs from colliding, while still allowing safe parallelism across sessions.

## Why

- Auto-reply runs can be expensive (LLM calls) and can collide when multiple inbound messages arrive close together.
- Serializing avoids competing for shared resources (session files, logs, CLI stdin) and reduces the chance of upstream rate limits.

## How it works

- A lane-aware FIFO queue drains each lane with a configurable concurrency cap (default 1 for unconfigured lanes; main defaults to 4, subagent to 8).
- `runEmbeddedPiAgent` enqueues by **session key** (lane `session:<key>`) to guarantee only one active run per session.
- Each session run is then queued into a **global lane** (`main` by default) so overall parallelism is capped by `agents.defaults.maxConcurrent`.
- When verbose logging is enabled, queued runs emit a short notice if they waited more than ~2s before starting.
- Typing indicators still fire immediately on enqueue (when supported by the channel) so user experience is unchanged while we wait our turn.

## Defaults

When unset, all inbound channel surfaces use:

- `mode: "steer"`
- `debounceMs: 500`
- `cap: 20`
- `drop: "summarize"`

`steer` is the default because it keeps the active model turn responsive without
starting a second session run. It drains all steering messages that arrived
before the next model boundary. If the current run cannot accept steering,
OpenClaw falls back to a followup queue entry.

## Queue modes

Inbound messages can steer the current run, wait for a followup turn, or do both:

- `steer`: queue steering messages into the active runtime. Pi delivers all pending steering messages **after the current assistant turn finishes executing its tool calls**, before the next LLM call; Codex app-server receives one batched `turn/steer`. If the run is not actively streaming or steering is unavailable, OpenClaw falls back to a followup queue entry.
- `queue` (legacy): old one-at-a-time steering. Pi delivers one queued steering message at each model boundary; Codex app-server receives separate `turn/steer` requests. Prefer `steer` unless you need the previous serialized behavior.
- `followup`: enqueue each message for a later agent turn after the current run ends.
- `collect`: coalesce queued messages into a **single** followup turn after the quiet window. If messages target different channels/threads, they drain individually to preserve routing.
- `steer-backlog` (aka `steer+backlog`): steer now **and** preserve the same message for a followup turn.
- `interrupt` (legacy): abort the active run for that session, then run the newest message.

Steer-backlog means you can get a followup response after the steered run, so
streaming surfaces can look like duplicates. Prefer `collect`/`steer` if you want
one response per inbound message.

For runtime-specific timing and dependency behavior, see
[Steering queue](/concepts/queue-steering). For the explicit `/steer <message>`
command, see [Steer](/tools/steer).

Configure globally or per channel via `messages.queue`:

```json5
{
  messages: {
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## Queue options

Options apply to `followup`, `collect`, and `steer-backlog` (and to `steer` or legacy `queue` when steering falls back to followup):

- `debounceMs`: quiet window before draining queued followups. Bare numbers are milliseconds; units `ms`, `s`, `m`, `h`, and `d` are accepted by `/queue` options.
- `cap`: max queued messages per session. Values below `1` are ignored.
- `drop: "summarize"`: default. Drop the oldest queued entries as needed, keep compact summaries, and inject them as a synthetic followup prompt.
- `drop: "old"`: drop the oldest queued entries as needed, without preserving summaries.
- `drop: "new"`: reject the newest message when the queue is already full.

Defaults: `debounceMs: 500`, `cap: 20`, `drop: summarize`.

## Precedence

For mode selection, OpenClaw resolves:

1. Inline or stored per-session `/queue` override.
2. `messages.queue.byChannel.<channel>`.
3. `messages.queue.mode`.
4. Default `steer`.

For options, inline or stored `/queue` options win over config. Then
channel-specific debounce (`messages.queue.debounceMsByChannel`), plugin
debounce defaults, global `messages.queue` options, and built-in defaults are
applied. `cap` and `drop` are global/session options, not per-channel config
keys.

## Per-session overrides

- Send `/queue <mode>` as a standalone command to store the mode for the current session.
- Options can be combined: `/queue collect debounce:0.5s cap:25 drop:summarize`
- `/queue default` or `/queue reset` clears the session override.

## Scope and guarantees

- Applies to auto-reply agent runs across all inbound channels that use the gateway reply pipeline (WhatsApp web, Telegram, Slack, Discord, Signal, iMessage, webchat, etc.).
- Default lane (`main`) is process-wide for inbound + main heartbeats; set `agents.defaults.maxConcurrent` to allow multiple sessions in parallel.
- Additional lanes may exist (e.g. `cron`, `cron-nested`, `nested`, `subagent`) so background jobs can run in parallel without blocking inbound replies. Isolated cron agent turns hold a `cron` slot while their inner agent execution uses `cron-nested`; both use `cron.maxConcurrentRuns`. Shared non-cron `nested` flows keep their own lane behavior. These detached runs are tracked as [background tasks](/automation/tasks).
- Per-session lanes guarantee that only one agent run touches a given session at a time.
- No external dependencies or background worker threads; pure TypeScript + promises.

## Troubleshooting

- If commands seem stuck, enable verbose logs and look for "queued for ...ms" lines to confirm the queue is draining.
- If you need queue depth, enable verbose logs and watch for queue timing lines.
- Codex app-server runs that accept a turn and then stop emitting progress are interrupted by the Codex adapter so the active session lane can release instead of waiting for the outer run timeout.
- When diagnostics are enabled, sessions that remain in `processing` past `diagnostics.stuckSessionWarnMs` with no observed reply, tool, status, block, or ACP progress are classified by current activity. Active work logs as `session.long_running`; active work with no recent progress logs as `session.stalled`; `session.stuck` is reserved for stale session bookkeeping with no active work, and only that path can release the affected session lane so queued work drains. Repeated `session.stuck` diagnostics back off while the session remains unchanged.

## Related

- [Session management](/concepts/session)
- [Steering queue](/concepts/queue-steering)
- [Steer](/tools/steer)
- [Retry policy](/concepts/retry)


## Queue Steering

When a message arrives while a session run is already streaming, OpenClaw can
send that message into the active runtime instead of starting another run for
the same session. The public modes are runtime-neutral; Pi and the native Codex
app-server harness implement the delivery details differently.

## Runtime boundary

Steering does not interrupt a tool call that is already running. Pi checks for
queued steering messages at model boundaries:

1. The assistant asks for tool calls.
2. Pi executes the current assistant message's tool-call batch.
3. Pi emits the turn end event.
4. Pi drains queued steering messages.
5. Pi appends those messages as user messages before the next LLM call.

This keeps tool results paired with the assistant message that requested them,
then lets the next model call see the latest user input.

The native Codex app-server harness exposes `turn/steer` instead of Pi's
internal steering queue. OpenClaw adapts the same modes there:

- `steer` batches queued messages for the configured quiet window, then sends a
  single `turn/steer` request with all collected user input in arrival order.
- `queue` keeps the legacy serialized shape by sending separate `turn/steer`
  requests.
- `followup`, `collect`, `steer-backlog`, and `interrupt` stay OpenClaw-owned
  queue behavior around the active Codex turn.

Codex review and manual compaction turns reject same-turn steering. When a
runtime cannot accept steering, OpenClaw falls back to the followup queue where
that mode allows it.

This page explains queue-mode steering for normal inbound messages. For the
explicit `/steer <message>` command, see [Steer](/tools/steer).

## Modes

| Mode            | Active-run behavior                                                                                                          | Later followup behavior                                                             |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `steer`         | Injects all queued steering messages together at the next runtime boundary. This is the default.                             | Falls back to followup only when steering is unavailable.                           |
| `queue`         | Legacy one-at-a-time steering. Pi injects one queued message per model boundary; Codex sends separate `turn/steer` requests. | Falls back to followup only when steering is unavailable.                           |
| `steer-backlog` | Same active-run steering behavior as `steer`.                                                                                | Also keeps the same message for a later followup turn.                              |
| `followup`      | Does not steer the current run.                                                                                              | Runs queued messages later.                                                         |
| `collect`       | Does not steer the current run.                                                                                              | Coalesces compatible queued messages into one later turn after the debounce window. |
| `interrupt`     | Aborts the active run, then starts the newest message.                                                                       | None.                                                                               |

## Burst example

If four users send messages while the agent is executing a tool call:

- `steer`: the active runtime receives all four messages in arrival order before
  its next model decision. Pi drains them at the next model boundary; Codex
  receives them as one batched `turn/steer`.
- `queue`: legacy serialized steering. Pi injects one queued message at a time;
  Codex receives separate `turn/steer` requests.
- `collect`: OpenClaw waits until the active run ends, then creates a followup
  turn with compatible queued messages after the debounce window.

## Scope

Steering always targets the current active session run. It does not create a new
session, change the active run's tool policy, or split messages by sender. In
multi-user channels, inbound prompts already include sender and route context, so
the next model call can see who sent each message.

Use `collect` when you want OpenClaw to build a later followup turn that can
coalesce compatible messages and preserve followup queue drop policy. Use
`queue` only when you need the older one-at-a-time steering behavior.

## Debounce

`messages.queue.debounceMs` applies to followup delivery, including `collect`,
`followup`, `steer-backlog`, and `steer` fallback when active-run steering is not
available. For Pi, active `steer` itself does not use the debounce timer because
Pi naturally batches messages until the next model boundary. For the native
Codex harness, OpenClaw uses the same debounce value as the quiet window before
sending the batched `turn/steer`.

## Related

- [Command queue](/concepts/queue)
- [Steer](/tools/steer)
- [Messages](/concepts/messages)
- [Agent loop](/concepts/agent-loop)

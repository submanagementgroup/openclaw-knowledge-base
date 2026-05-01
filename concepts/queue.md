---
domain: concepts
topic: "Command Queue and Queue Steering: Serializing Agent Turns, Queue Modes, and Message Handling"
type: concept
keywords:
  - command queue
  - queue steering
  - concurrent runs
  - collect mode
  - steer mode
  - followup mode
  - queue modes
related:
  - concepts/agent-loop
  - concepts/session
source:
  - concepts/queue.md
  - concepts/queue-steering.md
---

The command queue serializes agent turns per session, preventing concurrent run races. Queue steering controls how new messages interact with in-flight runs.

## Queue Modes

| Mode | Behavior |
|------|----------|
| `collect` | Queue incoming messages while agent runs, deliver all at end |
| `steer` | Queue but allow steering messages to interrupt the run |
| `followup` | Queue for delivery as a follow-up after current run completes |

We serialize inbound auto-reply runs (all channels) through a tiny in-process queue to prevent multiple agent runs from colliding, while still allowing safe parallelism across sessions.

## Why

- Auto-reply runs can be expensive (LLM calls) and can collide when multiple inbound messages arrive close together.
- Serializing avoids competing for shared resources (session files, logs, CLI stdin) and reduces the chance of upstream rate limits.

## How it works

- A lane-aware FIFO queue drains each lane with a configurable concurrency cap (default 1 for unconfigured lanes; main defaults to 4, subagent to 8).
- `runEmbeddedPiAgent` enqueues by **session key** (lane `session:<key>`) to guarantee only one active run per session.
- Each session run is then queued into a **global lane** (`main` by default) so overall parallelism is capped by `agents.defaults.maxConcurrent`.
- When verbose logging is enabled, queued runs emit a short notice if they waited more than ~2s before starting.
- Codex app-server runs that accept a turn and then stop emitting progress are interrupted by the Codex adapter so the active session lane can release instead of waiting for the outer run timeout.
- When diagnostics are enabled, sessions that remain in `processing` past `diagnostics.stuckSessionWarnMs` log a stuck-session warning. Active embedded runs, active reply operations, and active lane tasks remain warning-only by default; stale startup bookkeeping with no active session work can release the affected session lane so queued work drains.
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
[Steering queue](/concepts/queu

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

## Modes

| Mode            | Active-run behavior                                                                                                          | Later followup behavior                                                             |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `steer`         | Injects all queued steering messages together at the next runtime boundary. This is the default.                             | Falls back to followup only when steering is unavailable.                           |
| `queue`         | Legacy one-at-a-time steering. Pi injects one queued message per model boundary; Codex sends separate `turn/steer` requests. | Falls back to followup only when steering is unavailable.                           |
| `steer-backlog` | Same active-run steering behavior as `steer`.                                                                                | Also keeps the same message for a later followup turn.                              |
| `followup`      | Does not steer the current run.                                                                                              | Runs queued messages later.                                                         |
| `collect`       | Does not steer the current run.

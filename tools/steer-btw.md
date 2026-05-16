---
domain: tools
topic: "Steer and BTW Tools"
type: reference
keywords:
  - steer
  - steer tool
  - btw
  - by the way
  - agent guidance
source: 
  - tools/steer.md
  - tools/btw.md
---

## Steer Tool

`/steer` sends guidance to an already-active run. It is for "adjust this
run while it is still working" moments, not for starting a new turn.

## Current session

Use top-level `/steer` to target the active run for the current session:

```text
/steer prefer the smaller patch and keep the tests focused
/tell summarize before making the next tool call
```

Behavior:

- Targets only the current session's active run.
- Works independently of the session's `/queue` mode.
- Does not start a new run when the session is idle.
- Replies with a warning when there is no active run to steer.
- Uses the active runtime's steering path, so the model sees the guidance at
  the next supported runtime boundary.

## Steer vs queue

`/queue steer` changes how normal inbound messages behave when they arrive
while a run is active. `/steer <message>` is an explicit command that tries to
inject that command's message into the active run at the next supported runtime
boundary, regardless of the stored `/queue` setting.

Use:

- `/steer <message>` when you want to guide the active run right now.
- `/queue steer` when you want future normal messages to steer active runs by
  default.
- `/queue collect` or `/queue followup` when new messages should wait for a
  later turn instead of steering the active run.

For queue modes and fallback behavior, see [Command queue](/concepts/queue) and
[Steering queue](/concepts/queue-steering).

## Sub-agents

Use `/subagents steer` when the target is a child run:

```text
/subagents steer 2 focus only on the API surface
```

Top-level `/steer` does not select a sub-agent by id or list index. It always
targets the current session's active run. See [Sub-agents](/tools/subagents) for
sub-agent ids, labels, and control commands.

## ACP sessions

Use `/acp steer` when the target is an ACP harness session:

```text
/acp steer --session agent:main:acp:codex tighten the repro
```

See [ACP agents](/tools/acp-agents) for ACP session selection and runtime
behavior.

## Related

- [Slash commands](/tools/slash-commands)
- [Command queue](/concepts/queue)
- [Steering queue](/concepts/queue-steering)
- [Sub-agents](/tools/subagents)


## BTW Tool

`/btw` lets you ask a quick side question about the **current session** without
turning that question into normal conversation history. `/side` is an alias.

It is modeled after Claude Code's `/btw` behavior, but adapted to OpenClaw's
Gateway and multi-channel architecture.

## What it does

When you send:

```text
/btw what changed?
```

OpenClaw:

1. snapshots the current session context,
2. runs a separate ephemeral side query,
3. answers only the side question,
4. leaves the main run alone,
5. does **not** write the BTW question or answer to session history,
6. emits the answer as a **live side result** rather than a normal assistant message.

The important mental model is:

- same session context
- separate one-shot side query
- same native harness transport when the session uses a native harness
- no future context pollution
- no transcript persistence

For Codex harness sessions, BTW stays inside Codex by forking the active
app-server thread as an ephemeral side thread. That keeps Codex OAuth and native
thread behavior intact while still isolating the side answer from the parent
transcript. Like Codex `/side`, the side thread keeps the current Codex
permissions and native tool surface, with guardrails that tell the model not to
treat inherited parent-thread work as active instructions. Non-Codex runtimes
keep the older direct one-shot path.

## What it does not do

`/btw` does **not**:

- create a new durable session,
- continue the unfinished main task,
- write BTW question/answer data to transcript history,
- appear in `chat.history`,
- survive a reload.

It is intentionally **ephemeral**.

## How context works

BTW uses the current session as **background context only**.

If the main run is currently active, OpenClaw snapshots the current message
state and includes the in-flight main prompt as background context, while
explicitly telling the model:

- answer only the side question,
- do not resume or complete the unfinished main task,
- do not steer the parent conversation.

That keeps BTW isolated from the main run while still making it aware of what
the session is about.

## Delivery model

BTW is **not** delivered as a normal assistant transcript message.

At the Gateway protocol level:

- normal assistant chat uses the `chat` event
- BTW uses the `chat.side_result` event

This separation is intentional. If BTW reused the normal `chat` event path,
clients would treat it like regular conversation history.

Because BTW uses a separate live event and is not replayed from
`chat.history`, it disappears after reload.

## Surface behavior

### TUI

In TUI, BTW is rendered inline in the current session view, but it remains
ephemeral:

- visibly distinct from a normal assistant reply
- dismissible with `Enter` or `Esc`
- not replayed on reload

### External channels

On channels like Telegram, WhatsApp, and Discord, BTW is delivered as a
clearly labeled one-off reply because those surfaces do not have a local
ephemeral overlay concept.

The answer is still treated as a side result, not normal session history.

### Control UI / web

The Gateway emits BTW correctly as `chat.side_result`, and BTW is not included
in `chat.history`, so the persistence contract is already correct for web.

The current Control UI still needs a dedicated `chat.side_result` consumer to
render BTW live in the browser. Until that client-side support lands, BTW is a
Gateway-level feature with full TUI and external-channel behavior, but not yet
a complete browser UX.

## When to use BTW

Use `/btw` when you want:

- a quick clarification about the current work,
- a factual side answer while a long run is still in progress,
- a temporary answer that should not become part of future session context.

Examples:

```text
/btw what file are we editing?
/side what changed while the main run continued?
/btw what does this error mean?
/btw summarize the current task in one sentence
/btw what is 17 * 19?
```

## When not to use BTW

Do not use `/btw` when you want the answer to become part of the session's
future working context.

In that case, ask normally in the main session instead of using BTW.

## Related

**Slash commands:** Native command catalog and chat directives.
  

**Thinking levels:** Reasoning effort levels for the side-question model call.
  

**Session:** Session keys, history, and persistence semantics.
  

**Steer command:** Inject a steering message into the active run without ending it.
  


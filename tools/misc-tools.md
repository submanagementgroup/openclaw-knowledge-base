---
domain: tools
topic: "Miscellaneous Tools: btw, Capability Cookbook, Loop Detection, Trajectory, TokenJuice"
type: reference
keywords:
  - btw tool
  - capability cookbook
  - loop detection
  - trajectory
  - token optimization
  - TokenJuice
related:
  - tools/tools-overview
  - concepts/agent-loop
source:
  - tools/btw.md
  - tools/capability-cookbook.md
  - tools/loop-detection.md
  - tools/trajectory.md
---

Miscellaneous OpenClaw agent tools: btw (background notes), capability cookbook, loop detection, trajectory tracking, and token optimization.

## BTW (Background Note-Taking)

`/btw` lets you ask a quick side question about the **current session** without
turning that question into normal conversation history.

It is modeled after Claude Code's `/btw` behavior, but adapted to OpenClaw's
Gateway and multi-channel architecture.

## What it does

When you send:

```text
/btw what changed?
```

OpenClaw:

1. snapshots the current session context,
2. runs a separate **tool-less** model call,
3. answers only the side question,
4. leaves the main run alone,
5. does **not** write the BTW question or answer to session history,
6. emits the answer as a **live side result** rather than a normal assistant message.

The important mental model is:

- same session context
- separate one-shot side query
- no tool calls
- no future context pollution
- no transcript persistence

## What it does not do

`/btw` does **not**:

- create a new durable session,
- continue the unfinished main task,
- run tools or agent tool loops,
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
- do not emit tool calls or pseudo-tool calls.

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
/btw what does this error mean?
/btw summarize the current task in one sentence
/btw what is 17 * 19?
```

## When not to use BTW

Do not use `/btw` when you want the answer to become part of the session's
future working context.

In that case, ask normally in the main session instead of using BTW.

## Related

- [Slash commands](/tools/slash-commands)
- [Thinking Levels](/tools/thinking)
- [Session](/concepts/session)

## Capability Cookbook

This is a **contributor guide** for OpenClaw core developers. If you are
  building an external plugin, see [Building Plugins](/plugins/building-plugins)
  instead.

Use this when OpenClaw needs a new domain such as image generation, video
generation, or some future vendor-backed feature area.

The rule:

- plugin = ownership boundary
- capability = shared core contract

That means you should not start by wiring a vendor directly into a channel or a
tool. Start by defining the capability.

## When to create a capability

Create a new capability when all of these are true:

1. more than one vendor could plausibly implement it
2. channels, tools, or feature plugins should consume it without caring about
   the vendor
3. core needs to own fallback, policy, config, or delivery behavior

If the work is vendor-only and no shared contract exists yet, stop and define
the contract first.

## The standard sequence

1. Define the typed core contract.
2. Add plugin registration for that contract.
3. Add a shared runtime helper.
4. Wire one real vendor plugin as proof.
5. Move feature/channel consumers onto the runtime helper.
6. Add contract tests.
7. Document the operator-facing config and ownership model.

## What goes where

Core:

- request/response types
- provider registry + resolution
- fallback behavior
- config schema plus propagated `title` / `description` docs metadata on nested object, wildcard, array-item, and composition nodes
- runtime helper surface

Vendor plugin:

- vendor API calls
- vendor auth handling
- vendor-specific request normalization
- registration of the capability implementation

Feature/channel plugin:

- calls `api.runtime.*` or the matching `plugin-sdk/*-runtime` helper
- never calls a vendor implementation directly

## Provider and harness seams

Use provider hooks when the behavior belongs to the model provider contract
rather than the generic agent loop. Examples include provider-specific request
params after transport selection, auth-profil

## Loop Detection

OpenClaw can keep agents from getting stuck in repeated tool-call patterns.
The guard is **disabled by default**.

Enable it only where needed, because it can block legitimate repeated calls with strict settings.

## Why this exists

- Detect repetitive sequences that do not make progress.
- Detect high-frequency no-result loops (same tool, same inputs, repeated errors).
- Detect specific repeated-call patterns for known polling tools.

## Configuration block

Global defaults:

```json5
{
  tools: {
    loopDetection: {
      enabled: false,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

Per-agent override (optional):

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
            warningThreshold: 8,
            criticalThreshold: 16,
          },
        },
      },
    ],
  },
}
```

### Field behavior

- `enabled`: Master switch. `false` means no loop detection is performed.
- `historySize`: number of recent tool calls kept for analysis.
- `warningThreshold`: threshold before classifying a pattern as warning-only.
- `criticalThreshold`: threshold for blocking repetitive loop patterns.
- `globalCircuitBreakerThreshold`: global no-progress breaker threshold.
- `detectors.genericRepeat`: detects repeated same-tool + same-params patterns.
- `detectors.knownPollNoProgress`: detects known polling-like patterns with no state change.
- `detectors.pingPong`: detects alternating ping-pong patterns.

For `exec`, no-progress checks compare stable command outcomes and ignore volatile runtime metadata such as duration, PID, session ID, and working directory.
When a run id is available, recent tool-call history is evaluated only within that run so scheduled heartbeat cycles and fresh runs do not inherit stale loop counts from earlier runs.

## Recommended setup

- Start with `enabled: true`, defaults unchanged.
- Keep thresholds ordered as `warningThreshold < criticalThreshold < globalCircuitBreakerThreshold`.
- If false positives occur:
  - raise `warningThreshold` and/or `criticalThreshold`
  - (optionally) raise `globalCircuitBreakerThreshold`
  - disable only the detector causing issues
  - reduce `historySize` for less strict historical context

## Logs and expected behavior

When a loop is detected, OpenClaw reports a loop event and blocks or dampens the next tool-cycle depending on severity.
This protects users from runaway token spend and lockups while preserving normal tool access.

- Prefer warning and temporary suppression first.
- Escalate only when repeated evidence accumulates.

## Notes

- `tools.loopDetection` is merged with agent-level overrides.
- Per-agent config fully overrides or extends global values.
- If no config exists, guardrails stay off.

## Related

- [Exec approvals](/tools/exec-approvals)
- [Thinking levels](/tools/thinking)
- [Sub-agents](/tools/subagents)

## Trajectory Tracking

Trajectory capture is OpenClaw's per-session flight recorder. It records a
structured timeline for each agent run, then `/export-trajectory` packages the
current session into a redacted support bundle.

Use it when you need to answer questions like:

- What prompt, system prompt, and tools were sent to the model?
- Which transcript messages and tool calls led to this answer?
- Did the run time out, abort, compact, or hit a provider error?
- Which model, plugins, skills, and runtime settings were active?
- What usage and prompt-cache metadata did the provider return?

If you are filing a broad support report for a live Gateway issue, start with
[`/diagnostics`](/gateway/diagnostics#chat-command). Diagnostics collects the
sanitized Gateway bundle and, for OpenAI Codex harness sessions, can also send
Codex feedback to OpenAI servers after approval. Use `/export-trajectory` when
you specifically need the detailed per-session prompt, tool, and transcript
timeline.

## Quick start

Send this in the active session:

```text
/export-trajectory
```

Alias:

```text
/trajectory
```

OpenClaw writes the bundle under the workspace:

```text
.openclaw/trajectory-exports/openclaw-trajectory-<session>-<timestamp>/
```

You can choose a relative output directory name:

```text
/export-trajectory bug-1234
```

The custom path is resolved inside `.openclaw/trajectory-exports/`. Absolute
paths and `~` paths are rejected.

Trajectory bundles can contain prompts, model messages, tool schemas, tool
results, runtime events, and local paths. The chat slash command therefore runs
through exec approval every time. Approve the export once when you intend to
create the bundle; do not use allow-all. In group chats, OpenClaw sends the
approval prompt and export result to the owner privately instead of posting the
trajectory details back to the shared room.

For local inspection or support workflows, you can also run the approved command
path directly:

```bash
openclaw sessions export-trajectory

## TokenJuice (Token Optimization)

`tokenjuice` is an optional bundled plugin that compacts noisy `exec` and `bash`
tool results after the command has already run.

It changes the returned `tool_result`, not the command itself. Tokenjuice does
not rewrite shell input, rerun commands, or change exit codes.

Today this applies to PI embedded runs and OpenClaw dynamic tools in the Codex
app-server harness. Tokenjuice hooks OpenClaw's tool-result middleware and
trims the output before it goes back into the active harness session.

## Enable the plugin

Fast path:

```bash
openclaw config set plugins.entries.tokenjuice.enabled true
```

Equivalent:

```bash
openclaw plugins enable tokenjuice
```

OpenClaw already ships the plugin. There is no separate `plugins install`
or `tokenjuice install openclaw` step.

If you prefer editing config directly:

```json5
{
  plugins: {
    entries: {
      tokenjuice: {
        enabled: true,
      },
    },
  },
}
```

## What tokenjuice changes

- Compacts noisy `exec` and `bash` results before they are fed back into the session.
- Keeps the original command execution untouched.
- Preserves exact file-content reads and other commands that tokenjuice should leave raw.
- Stays opt-in: disable the plugin if you want verbatim output everywhere.

## Verify it is working

1. Enable the plugin.
2. Start a session that can call `exec`.
3. Run a noisy command such as `git status`.
4. Check that the returned tool result is shorter and more structured than the raw shell output.

## Disable the plugin

```bash
openclaw config set plugins.entries.tokenjuice.enabled false
```

Or:

```bash
openclaw plugins disable tokenjuice
```

## Related

- [Exec tool](/tools/exec)
- [Thinking levels](/tools/thinking)
- [Context engine](/concepts/context-engine)

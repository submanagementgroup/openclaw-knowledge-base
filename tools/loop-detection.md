---
domain: tools
topic: "Loop Detection and Agent Trajectory"
type: concept
keywords:
  - loop detection
  - trajectory
  - agent loops
  - stuck detection
  - agent trajectory
source: 
  - tools/loop-detection.md
  - tools/trajectory.md
---

## Loop Detection

OpenClaw has two cooperating guardrails for repetitive tool-call patterns:

1. **Loop detection** (`tools.loopDetection.enabled`) — disabled by default. Watches the rolling tool-call history for repeated patterns and unknown-tool retries.
2. **Post-compaction guard** (`tools.loopDetection.postCompactionGuard`) — enabled by default unless `tools.loopDetection.enabled` is explicitly `false`. Arms after every compaction-retry and aborts the run when the agent emits the same `(tool, args, result)` triple within the window.

Both are configured under the same `tools.loopDetection` block, but the post-compaction guard runs whenever the master switch is not explicitly off. Set `tools.loopDetection.enabled: false` to silence both surfaces.

## Why this exists

- Detect repetitive sequences that do not make progress.
- Detect high-frequency no-result loops (same tool, same inputs, repeated errors).
- Detect specific repeated-call patterns for known polling tools.
- Prevent context-overflow then compaction then same-loop cycles from running indefinitely.

## Configuration block

Global defaults, with every documented field shown:

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // master switch for the rolling-history detectors
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      unknownToolThreshold: 10,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
      postCompactionGuard: {
        windowSize: 3, // armed after compaction-retry; runs unless enabled is explicitly false
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

| Field                            | Default | Effect                                                                                                                          |
| -------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                        | `false` | Master switch for the rolling-history detectors. Setting `false` also disables the post-compaction guard.                       |
| `historySize`                    | `30`    | Number of recent tool calls kept for analysis.                                                                                  |
| `warningThreshold`               | `10`    | Threshold before a pattern is classified as warning-only.                                                                       |
| `criticalThreshold`              | `20`    | Threshold for blocking repetitive no-progress loop patterns.                                                                    |
| `unknownToolThreshold`           | `10`    | Block repeated calls to the same unavailable tool after this many misses.                                                       |
| `globalCircuitBreakerThreshold`  | `30`    | Global no-progress breaker threshold across all detectors.                                                                      |
| `detectors.genericRepeat`        | `true`  | Warns on repeated same-tool + same-params patterns and blocks when the same calls also return identical outcomes.               |
| `detectors.knownPollNoProgress`  | `true`  | Detects known polling-like patterns with no state change.                                                                       |
| `detectors.pingPong`             | `true`  | Detects alternating ping-pong patterns.                                                                                         |
| `postCompactionGuard.windowSize` | `3`     | Number of post-compaction tool calls during which the guard stays armed and the count of identical triples that aborts the run. |

For `exec`, no-progress checks compare stable command outcomes and ignore volatile runtime metadata such as duration, PID, session ID, and working directory. When a run id is available, recent tool-call history is evaluated only within that run so scheduled heartbeat cycles and fresh runs do not inherit stale loop counts from earlier runs.

## Recommended setup

- For smaller models, set `enabled: true` and leave the thresholds at their defaults. Flagship models rarely need rolling-history detection and can leave the master switch at `false` while still benefiting from the post-compaction guard.
- Keep thresholds ordered as `warningThreshold < criticalThreshold < globalCircuitBreakerThreshold`.
- If false positives occur:
  - Raise `warningThreshold` and/or `criticalThreshold`.
  - Optionally raise `globalCircuitBreakerThreshold`.
  - Disable only the specific detector causing issues (`detectors.<name>: false`).
  - Reduce `historySize` for less strict historical context.
- To disable everything (including the post-compaction guard), set `tools.loopDetection.enabled: false` explicitly.

## Post-compaction guard

When the runner completes a compaction-retry after a context-overflow, it arms a short-window guard that watches the next few tool calls. If the agent emits the same `(toolName, argsHash, resultHash)` triple multiple times within the window, the guard concludes that compaction did not break the loop and aborts the run with a `compaction_loop_persisted` error.

The guard is gated by the master `tools.loopDetection.enabled` flag with one twist: it stays **enabled when the flag is unset or `true`** and only deactivates when the flag is explicitly `false`. This is intentional. The guard exists to escape compaction loops that would otherwise burn unbounded tokens, so a no-config user still gets the protection.

```json5
{
  tools: {
    loopDetection: {
      // master switch; set false to disable the guard along with the rolling detectors
      enabled: true,
      postCompactionGuard: {
        windowSize: 3, // default
      },
    },
  },
}
```

- Lower `windowSize` is stricter (fewer attempts before abort).
- Higher `windowSize` gives the agent more recovery attempts.
- The guard never aborts when results are changing, only when results are byte-identical across the window.
- It is intentionally narrow: it fires only in the immediate aftermath of a compaction-retry.

> **Note:** The post-compaction guard runs whenever the master flag is not explicitly `false`, even if you never wrote a `tools.loopDetection` block. To verify, look for `post-compaction guard armed for N attempts` in the gateway log immediately after a compaction event.


## Logs and expected behavior

When a loop is detected, OpenClaw reports a loop event and either dampens or blocks the next tool-cycle depending on severity. This protects users from runaway token spend and lockups while preserving normal tool access.

- Warnings come first.
- Suppression follows when patterns persist past the warning threshold.
- Critical thresholds block the next tool-cycle and surface a clear loop-detection reason in the run record.
- The post-compaction guard emits `compaction_loop_persisted` errors with the offending tool name and identical-call count.

## Related

**Exec approvals:** Allow/deny policy for shell execution.
  

**Thinking levels:** Reasoning effort levels and provider-policy interaction.
  

**Sub-agents:** Spawning isolated agents to bound runaway behavior.
  

**Configuration reference:** Full `tools.loopDetection` schema and merging semantics.
  



## Agent Trajectory

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
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
```

## Access

Trajectory export is an owner command. The sender must pass the normal command
authorization checks and owner checks for the channel.

## What gets recorded

Trajectory capture is on by default for OpenClaw agent runs.

Runtime events include:

- `session.started`
- `trace.metadata`
- `context.compiled`
- `prompt.submitted`
- `model.fallback_step`, including the source model, next model, failure reason/detail, chain position, and whether fallback advanced, succeeded, or exhausted the chain
- `model.completed`
- `trace.artifacts`
- `session.ended`

Transcript events are also reconstructed from the active session branch:

- user messages
- assistant messages
- tool calls
- tool results
- compactions
- model changes
- labels and custom session entries

Events are written as JSON Lines with this schema marker:

```json
{
  "traceSchema": "openclaw-trajectory",
  "schemaVersion": 1
}
```

## Bundle files

An exported bundle can contain:

| File                  | Contents                                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| `manifest.json`       | Bundle schema, source files, event counts, and generated file list                             |
| `events.jsonl`        | Ordered runtime and transcript timeline                                                        |
| `session-branch.json` | Redacted active transcript branch and session header                                           |
| `metadata.json`       | OpenClaw version, OS/runtime, model, config snapshot, plugins, skills, and prompt metadata     |
| `artifacts.json`      | Final status, errors, usage, prompt cache, compaction count, assistant text, and tool metadata |
| `prompts.json`        | Submitted prompts and selected prompt-building details                                         |
| `system-prompt.txt`   | Latest compiled system prompt, when captured                                                   |
| `tools.json`          | Tool definitions sent to the model, when captured                                              |

`manifest.json` lists the files present in that bundle. Some files are omitted
when the session did not capture the corresponding runtime data.

## Capture location

By default, runtime trajectory events are written beside the session file:

```text
<session>.trajectory.jsonl
```

OpenClaw also writes a best-effort pointer file beside the session:

```text
<session>.trajectory-path.json
```

Set `OPENCLAW_TRAJECTORY_DIR` to store runtime trajectory sidecars in a
dedicated directory:

```bash
export OPENCLAW_TRAJECTORY_DIR=/var/lib/openclaw/trajectories
```

When this variable is set, OpenClaw writes one JSONL file per session id in that
directory.

Session maintenance removes trajectory sidecars when their owning session entry
is pruned, capped, or evicted by the sessions disk budget. Runtime files outside
the sessions directory are removed only when the pointer target still proves it
belongs to that session.

## Disable capture

Set `OPENCLAW_TRAJECTORY=0` before starting OpenClaw:

```bash
export OPENCLAW_TRAJECTORY=0
```

This disables runtime trajectory capture. `/export-trajectory` can still export
the transcript branch, but runtime-only files such as compiled context,
provider artifacts, and prompt metadata may be missing.

## Privacy and limits

Trajectory bundles are designed for support and debugging, not public posting.
OpenClaw redacts sensitive values before writing export files:

- credentials and known secret-like payload fields
- image data
- local state paths
- workspace paths, replaced with `$WORKSPACE_DIR`
- home directory paths, where detected

The exporter also bounds input size:

- runtime sidecar files: live capture stops at 10 MiB and records a truncation event when space remains; export accepts existing runtime sidecars up to 50 MiB
- session files: 50 MiB
- runtime events: 200,000
- total exported events: 250,000
- individual runtime event lines are truncated above 256 KiB

Review bundles before sharing them outside your team. Redaction is best-effort
and cannot know every application-specific secret.

## Troubleshooting

If the export has no runtime events:

- confirm OpenClaw was started without `OPENCLAW_TRAJECTORY=0`
- check whether `OPENCLAW_TRAJECTORY_DIR` points to a writable directory
- run another message in the session, then export again
- inspect `manifest.json` for `runtimeEventCount`

If the command rejects the output path:

- use a relative name like `bug-1234`
- do not pass `/tmp/...` or `~/...`
- keep the export inside `.openclaw/trajectory-exports/`

If the export fails with a size error, the session or sidecar exceeded the
export safety limits. Start a new session or export a smaller reproduction.

## Related

- [Diffs](/tools/diffs)
- [Session management](/concepts/session)
- [Exec tool](/tools/exec)

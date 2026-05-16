---
domain: internals
topic: "Plan: Codex Context Engine Harness"
type: concept
keywords:
  - codex-context-engine-harness
  - planning
  - design
  - roadmap
source: plan/codex-context-engine-harness.md
---

## Status

Draft implementation specification.

## Goal

Make the bundled Codex app-server harness honor the same OpenClaw context-engine
lifecycle contract that embedded PI turns already honor.

A session using `agents.defaults.embeddedHarness.runtime: "codex"` or a
`codex/*` model should still let the selected context-engine plugin, such as
`lossless-claw`, control context assembly, post-turn ingest, maintenance, and
OpenClaw-level compaction policy as far as the Codex app-server boundary allows.

## Non-goals

- Do not reimplement Codex app-server internals.
- Do not make Codex native thread compaction produce a lossless-claw summary.
- Do not require non-Codex models to use the Codex harness.
- Do not change ACP/acpx session behavior. This specification is for the
  non-ACP embedded agent harness path only.
- Do not make third-party plugins register Codex app-server extension factories;
  the existing bundled-plugin trust boundary remains unchanged.

## Current architecture

The embedded run loop resolves the configured context engine once per run before
selecting a concrete low-level harness:

- `src/agents/pi-embedded-runner/run.ts`
  - initializes context-engine plugins
  - calls `resolveContextEngine(params.config)`
  - passes `contextEngine` and `contextTokenBudget` into
    `runEmbeddedAttemptWithBackend(...)`

`runEmbeddedAttemptWithBackend(...)` delegates to the selected agent harness:

- `src/agents/pi-embedded-runner/run/backend.ts`
- `src/agents/harness/selection.ts`

The Codex app-server harness is registered by the bundled Codex plugin:

- `extensions/codex/index.ts`
- `extensions/codex/harness.ts`

The Codex harness implementation receives the same `EmbeddedRunAttemptParams`
as PI-backed attempts:

- `extensions/codex/src/app-server/run-attempt.ts`

That means the required hook point is in OpenClaw-controlled code. The external
boundary is the Codex app-server protocol itself: OpenClaw can control what it
sends to `thread/start`, `thread/resume`, and `turn/start`, and can observe
notifications, but it cannot change Codex's internal thread store or native
compactor.

## Current gap

Embedded PI attempts call the context-engine lifecycle directly:

- bootstrap/maintenance before the attempt
- assemble before the model call
- afterTurn or ingest after the attempt
- maintenance after a successful turn
- context-engine compaction for engines that own compaction

Relevant PI code:

- `src/agents/pi-embedded-runner/run/attempt.ts`
- `src/agents/pi-embedded-runner/run/attempt.context-engine-helpers.ts`
- `src/agents/pi-embedded-runner/context-engine-maintenance.ts`

Codex app-server attempts currently run generic agent-harness hooks and mirror
the transcript, but do not call `params.contextEngine.bootstrap`,
`params.contextEngine.assemble`, `params.contextEngine.afterTurn`,
`params.contextEngine.ingestBatch`, `params.contextEngine.ingest`, or
`params.contextEngine.maintain`.

Relevant Codex code:

- `extensions/codex/src/app-server/run-attempt.ts`
- `extensions/codex/src/app-server/thread-lifecycle.ts`
- `extensions/codex/src/app-server/event-projector.ts`
- `extensions/codex/src/app-server/compact.ts`

## Desired behavior

For Codex harness turns, OpenClaw should preserve this lifecycle:

1. Read the mirrored OpenClaw session transcript.
2. Bootstrap the active context engine when a previous session file exists.
3. Run bootstrap maintenance when available.
4. Assemble context using the active context engine.
5. Convert the assembled context into Codex-compatible inputs.
6. Start or resume the Codex thread with developer instructions that include any
   context-engine `systemPromptAddition`.
7. Start the Codex turn with the assembled user-facing prompt.
8. Mirror the Codex result back into the OpenClaw transcript.
9. Call `afterTurn` if implemented, otherwise `ingestBatch`/`ingest`, using the
   mirrored transcript snapshot.
10. Run turn maintenance after successful non-aborted turns.
11. Preserve Codex native compaction signals and OpenClaw compaction hooks.

## Design constraints

### Codex app-server remains canonical for native thread state

Codex owns its native thread and any internal extended history. OpenClaw should
not try to mutate the app-server's internal history except through supported
protocol calls.

OpenClaw's transcript mirror remains the source for OpenClaw features:

- chat history
- search
- `/new` and `/reset` bookkeeping
- future model or harness switching
- context-engine plugin state

### Context engine assembly must be projected into Codex inputs

The context-engine interface returns OpenClaw `AgentMessage[]`, not a Codex
thread patch. Codex app-server `turn/start` accepts a current user input, while
`thread/start` and `thread/resume` accept developer instructions.

Therefore the implementation needs a projection layer. The safe first version
should avoid pretending it can replace Codex internal history. It should inject
assembled context as deterministic prompt/developer-instruction material around
the current turn.

### Prompt-cache stability matters

For engines like lossless-claw, the assembled context should be deterministic
for unchanged inputs. Do not add timestamps, random ids, or nondeterministic
ordering to generated context text.

### Runtime selection semantics do not change

Harness selection remains as-is:

- `runtime: "pi"` forces PI
- `runtime: "codex"` selects the registered Codex harness
- `runtime: "auto"` lets plugin harnesses claim supported providers
- unmatched `auto` runs use PI

This work changes what happens after the Codex harness is selected.

## Implementation plan

### 1. Export or relocate reusable context-engine attempt helpers

Today the reusable lifecycle helpers live under the PI runner:

- `src/agents/pi-embedded-runner/run/attempt.context-engine-helpers.ts`
- `src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts`
- `src/agents/pi-embedded-runner/context-engine-maintenance.ts`

Codex should not import from an implementation path whose name implies PI if we
can avoid it.

Create a harness-neutral module, for example:

- `src/agents/harness/context-engine-lifecycle.ts`

Move or re-export:

- `runAttemptContextEngineBootstrap`
- `assembleAttemptContextEngine`
- `finalizeAttemptContextEngineTurn`
- `buildAfterTurnRuntimeContext`
- `buildAfterTurnRuntimeContextFromUsage`
- a small wrapper around `runContextEngineMaintenance`

Keep PI imports working either by re-exporting from the old files or updating PI
call sites in the same PR.

The neutral helper names should not mention PI.

Suggested names:

- `bootstrapHarnessContextEngine`
- `assembleHarnessContextEngine`
- `finalizeHarnessContextEngineTurn`
- `buildHarnessContextEngineRuntimeContext`
- `runHarnessContextEngineMaintenance`

### 2. Add a Codex context projection helper

Add a new module:

- `extensions/codex/src/app-server/context-engine-projection.ts`

Responsibilities:

- Accept the assembled `AgentMessage[]`, original mirrored history, and current
  prompt.
- Determine which context belongs in developer instructions vs current user
  input.
- Preserve the current user prompt as the final actionable request.
- Render prior messages in a stable, explicit format.
- Avoid volatile metadata.

Proposed API:

```ts
export type CodexContextProjection = {
  developerInstructionAddition?: string;
  promptText: string;
  assembledMessages: AgentMessage[];
  prePromptMessageCount: number;
};

export function projectContextEngineAssemblyForCodex(params: {
  assembledMessages: AgentMessage[];
  originalHistoryMessages: AgentMessage[];
  prompt: string;
  systemPromptAddition?: string;
}): CodexContextProjection;
```

Recommended first projection:

- Put `systemPromptAddition` into developer instructions.
- Put the assembled transcript context before the current prompt in `promptText`.
- Label it clearly as OpenClaw assembled context.
- Keep current prompt last.
- Exclude duplicate current user prompt if it already appears at the tail.

Example prompt shape:

```text
OpenClaw assembled context for this turn:

<conversation_context>
[user]
...

[assistant]
...
</conversation_context>

Current user request:
...
```

This is less elegant than native Codex history surgery, but it is implementable
inside OpenClaw and preserves context-engine semantics.

Future improvement: if Codex app-server exposes a protocol for replacing or
supplementing thread history, swap this projection layer to use that API.

### 3. Wire bootstrap before Codex thread startup

In `extensions/codex/src/app-server/run-attempt.ts`:

- Read mirrored session history as today.
- Determine whether the session file existed before this run. Prefer a helper
  that checks `fs.stat(params.sessionFile)` before mirroring writes.
- Open a `SessionManager` or use a narrow session manager adapter if the helper
  requires it.
- Call the neutral bootstrap helper when `params.contextEngine` exists.

Pseudo-flow:

```ts
const hadSessionFile = await fileExists(params.sessionFile);
const sessionManager = SessionManager.open(params.sessionFile);
const historyMessages = sessionManager.buildSessionContext().messages;

await bootstrapHarnessContextEngine({
  hadSessionFile,
  contextEngine: params.contextEngine,
  sessionId: params.sessionId,
  sessionKey: sandboxSessionKey,
  sessionFile: params.sessionFile,
  sessionManager,
  runtimeContext: buildHarnessContextEngineRuntimeContext(...),
  runMaintenance: runHarnessContextEngineMaintenance,
  warn,
});
```

Use the same `sessionKey` convention as the Codex tool bridge and transcript
mirror. Today Codex computes `sandboxSessionKey` from `params.sessionKey` or
`params.sessionId`; use that consistently unless there is a reason to preserve
raw `params.sessionKey`.

### 4. Wire assemble before `thread/start` / `thread/resume` and `turn/start`

In `runCodexAppServerAttempt`:

1. Build dynamic tools first, so the context engine sees the actual available
   tool names.
2. Read mirrored session history.

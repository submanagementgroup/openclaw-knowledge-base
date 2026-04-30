---
domain: internals
topic: "Design Plans: Codex Context Engine Harness and UI Channels Architecture"
type: reference
keywords:
  - design plans
  - architecture plans
  - context engine design
  - UI channels design
related:
  - concepts/context-engine
  - plugins/codex-harness
source:
  - plan/codex-context-engine-harness.md
  - plan/ui-channels.md
---

OpenClaw design and planning documents.

## Codex Context Engine Harness Plan

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
assembled context as deterministic prompt/developer

## UI Channels Plan

## Status

Implemented for the shared agent, CLI, plugin capability, and outbound delivery surfaces:

- `ReplyPayload.presentation` carries semantic message UI.
- `ReplyPayload.delivery.pin` carries sent-message pin requests.
- Shared message actions expose `presentation`, `delivery`, and `pin` instead of provider-native `components`, `blocks`, `buttons`, or `card`.
- Core renders or auto-degrades presentation through plugin-declared outbound capabilities.
- Discord, Slack, Telegram, Mattermost, MS Teams, and Feishu renderers consume the generic contract.
- Discord channel control-plane code no longer imports Carbon-backed UI containers.

Canonical docs now live in [Message Presentation](/plugins/message-presentation).
Keep this plan as historical implementation context; update the canonical guide
for contract, renderer, or fallback behavior changes.

## Problem

Channel UI is currently split across several incompatible surfaces:

- Core owns a Discord-shaped cross-context renderer hook through `buildCrossContextComponents`.
- Discord `channel.ts` can import native Carbon UI through `DiscordUiContainer`, which pulls runtime UI dependencies into the channel plugin control plane.
- The agent and CLI expose native payload escape hatches such as Discord `components`, Slack `blocks`, Telegram or Mattermost `buttons`, and Teams or Feishu `card`.
- `ReplyPayload.channelData` carries both transport hints and native UI envelopes.
- The generic `interactive` model exists, but it is narrower than the richer layouts already used by Discord, Slack, Teams, Feishu, LINE, Telegram, and Mattermost.

This makes core aware of native UI shapes, weakens plugin runtime laziness, and gives agents too many provider-specific ways to express the same message intent.

## Goals

- Core decides the best semantic presentation for a message from declared capabilities.
- Extensions declare capabilities and render semantic presentation into native transport payloads.
- Web Control UI remains separate from chat native UI.
- Native channel payloads are not exposed through the shared agent or CLI message surface.
- Unsupported presentation features auto-degrade to the best text representation.
- Delivery behavior such as pinning a sent message is generic delivery metadata, not presentation.

## Non goals

- No backwards compatibility shim for `buildCrossContextComponents`.
- No public native escape hatches for `components`, `blocks`, `buttons`, or `card`.
- No core imports of channel-native UI libraries.
- No provider-specific SDK seams for bundled channels.

## Target model

Add a core-owned `presentation` field to `ReplyPayload`.

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

`interactive` becomes a subset of `presentation` during migration:

- `interactive` text block maps to `presentation.blocks[].type = "text"`.
- `interactive` buttons block maps to `presentation.blocks[].type = "buttons"`.
- `interactive` select block maps to `presentation.blocks[].type = "select"`.

The external agent and CLI schemas now use `presentation`; `interactive` remains an internal legacy parser/rendering helper for existing reply producers.

## Delivery metadata

Add a core-owned `delivery` field for send behavior that is not UI.

```ts
type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;

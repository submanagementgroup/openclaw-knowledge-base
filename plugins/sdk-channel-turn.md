---
domain: plugins
topic: "SDK Channel Turn: Inbound Message Handling, Turn Context, and Reply Delivery"
type: reference
keywords:
  - channel turn
  - turn handling
  - inbound messages
  - reply delivery
  - turn context
  - SDK turn
related:
  - plugins/sdk-channel-plugins
  - plugins/sdk-overview
source: plugins/sdk-channel-turn.md
---

Channel turn handling in the OpenClaw SDK: how the SDK manages inbound message turns and reply delivery.

The channel turn kernel is the shared inbound state machine that turns a normalized platform event into an agent turn. Channel plugins provide the platform facts and the delivery callback. Core owns the orchestration: ingest, classify, preflight, resolve, authorize, assemble, record, dispatch, and finalize.

Use this when your plugin is on the inbound message hot path. For non-message events (slash commands, modals, button interactions, lifecycle events, reactions, voice state), keep them plugin-local. The kernel only owns events that may become an agent text turn.

  The kernel is reached through the injected plugin runtime as `runtime.channel.turn.*`. The plugin runtime type is exported from `openclaw/plugin-sdk/core`, so third-party native plugins can use these entry points the same way bundled channel plugins do.

## Why a shared kernel

Channel plugins repeat the same inbound flow: normalize, route, gate, build a context, record session metadata, dispatch the agent turn, finalize delivery state. Without a shared kernel, a change to mention gating, tool-only visible replies, session metadata, pending history, or dispatch finalization has to be applied per channel.

The kernel keeps four concepts deliberately separate:

- `ConversationFacts`: where the message came from
- `RouteFacts`: which agent and session should process it
- `ReplyPlanFacts`: where visible replies should go
- `MessageFacts`: what body and supplemental context the agent should see

Slack DMs, Telegram topics, Matrix threads, and Feishu topic sessions all distinguish these in practice. Treating them as one identifier causes drift over time.

## Stage lifecycle

The kernel runs the same fixed pipeline regardless of channel:

1. `ingest` -- adapter converts a raw platform event into `NormalizedTurnInput`
2. `classify` -- adapter declares whether this event can start an agent turn
3. `preflight` -- adapter does dedupe, self-echo, hydration, debounce, decryption, partial fact prefill
4. `resolve` -- adapter returns a fully assembled turn (route, reply plan, message, delivery)
5. `authorize` -- DM, group, mention, and command policy applied to the assembled facts
6. `assemble` -- `FinalizedMsgContext` built from the facts via `buildContext`
7. `record` -- inbound session metadata and last route persisted
8. `dispatch` -- agent turn executed through the buffered block dispatcher
9. `finalize` -- adapter `onFinalize` runs even on dispatch error

Each stage emits a structured log event when a `log` callback is supplied. See [Observability](#observability).

## Admission kinds

The kernel does not throw when a turn is gated. It returns a `ChannelTurnAdmission`:

| Kind          | When                                                                                                                                         |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `dispatch`    | Turn is admitted. Agent turn runs and the visible reply path is exercised.                                                                   |
| `observeOnly` | Turn runs end-to-end but the delivery adapter sends nothing visible. Used for broadcast observer agents and other passive multi-agent flows. |
| `handled`     | A platform event was consumed locally (lifecycle, reaction, button, modal). Kernel skips dispatch.                                           |
| `drop`        | Skip path. Optionally `recordHistory: true` keeps the message in pending group history so a future mention has context.                      |

Admission can come from `classify` (event class said it cannot start a turn), from `preflight` (dedupe, self-echo, missing mention with history record), or from `resolveTurn` itself.

## Entry points

The runtime exposes three preferred entry points so adapters can opt in at the level that matches the channel.

```typescript
runtime.channel.turn.run(...)             // adapter-driven full pipeline
runtime.channel.turn.runPrepared(...)     // channel owns dispatch; kernel runs record + finalize
runtime.channel.turn.buildContext(...)    // pure facts to FinalizedMsgContext mapping
```

Two older runtime helpers remain available for Plugin SDK compatibility:

```typescript
runtime.channel.turn.runResolved(...)      // deprecated compatibility alias; prefer run
runtime.channel.turn.dispatchAssembled(...) // deprecated compatibility alias; prefer run or runPrepared
```

### run

Use when your channel can express its inbound flow as a `ChannelTurnAdapter<TRaw>`. The adapter has callbacks for `ingest`, optional `classify`, optional `preflight`, mandatory `resolveTurn`, and optional `onFinalize`.

```typescript
await runtime.channel.turn.run({
  channel: "tlon",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest(raw) {
      return {
        id: raw.messageId,
        timestamp: raw.timestamp,
        rawText: raw.body,
        textForAgent: raw.body,
      };
    },
    classify(input) {
      return { kind: "message", canStartAgentTurn: input.rawText.length > 0 };
    },
    async preflight(input, eventClass) {
      if (await isDuplicate(input.id)) {
        return { admission: { kind: "drop", reason: "dedupe" } };
      }
      return {};
    },
    resolveTurn(input) {
      return buildAssembledTurn(input);
    },
    onFinalize(result) {
      clearPendingGroupHistory(result);
    },
  },
});
```

`run` is the right shape when the channel has small adapter logic and benefits from owning the lifecycle through hooks.

### runPrepared

Use when the channel has a complex local dispatcher with previews, retries, edits, or thread bootstrap that must stay channel-owned. The kernel still records the inbound session before dispatch and surfaces a uniform `DispatchedChannelTurnResult`.

```typescript
const { dispatchResult } = await runtime.channel.turn.runPrepared({
  channel: "matrix",
  accountId,
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  record: {
    onRecordError,
    updateLastRoute,
  },
  onPreDispatchFailure: async (err) => {
    await stopStatusReactions();
  },
  runDispatch: async () => {
    return await runMatrixOwnedDispatcher();
  },
});
```

Rich channels (Matrix, Mattermost, Microsoft Teams, Feishu, QQ Bot) use `runPrepared` because their dispatcher orchestrates platform-specific behavior the kernel must not learn about.

### buildContext

A pure function that maps fact bundles into `FinalizedMsgContext`. Use it when your channel hand-rolls part of the pipeline but wants consistent context shape.

```typescript
const ctxPayload = runtime.channel.turn.buildContext({
  channel: "googlechat",
  accountId,
  messageId,
  timestamp,
  from,
  sender,
  conversation,
  route,
  reply,
  message,
  access,
  media,
  supplemental,
});
```

`buildContext` is also useful inside `resolveTurn` callbacks when assembling a turn for `run`.

  Deprecated SDK helpers such as `dispatchInboundReplyWithBase` still bridge through an assembled-turn helper. New plugin code should use `run` or `runPrepared`.

## Fact types

The facts the kernel consumes from your adapter are platform-agnostic. Translate platform objects into these shapes before handing them to the kernel.

### NormalizedTurnInput

| Field             | Purpose                                                                      |
| ----------------- | ---------------------------------------------------------------------------- |
| `id`              | Stable message id used for dedupe and logs                                   |
| `timestamp`       | Optional epoch ms                                                            |
| `rawText`         | Body as received from the platform                                           |
| `textForAgent`    | Optional cleaned body for the agent (mention strip, typing trim)             |
| `textForCommands` | Optional body used for `/command` parsing                                    |
| `raw`             | Optional pass-through reference for adapter callbacks that need the original |

### ChannelEventClass

| Field                  | Purpose                                                                 |
| ---------------------- | ----------------------------------------------------------------------- |
| `kind`                 | `message`, `command`, `interaction`, `reaction`, `lifecycle`, `unknown` |
| `canStartAgentTurn`    | If false the kernel returns `{ kind: "handled" }`                       |
| `requiresImmediateAck` | Hint for adapters that need to ACK before dispatch                      |

### SenderFacts

| Field          | Purpose                                                        |
| -------------- | -------------------------------------------------------------- |
| `id`           | Stable platform sender id                                      |
| `name`         | Display name

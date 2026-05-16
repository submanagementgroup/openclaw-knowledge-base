---
domain: internals
topic: "Message Lifecycle Refactor Design Doc"
type: concept
keywords:
  - message lifecycle
  - refactor
  - message processing
  - turn pipeline
  - internal design
source: concepts/message-lifecycle-refactor.md
---

This page is the target design for replacing scattered channel turn, reply
dispatch, preview streaming, and outbound delivery helpers with one durable
message lifecycle.

The short version:

- The core primitives should be **receive** and **send**, not **reply**.
- A reply is only a relation on an outbound message.
- A turn is an inbound-processing convenience, not the owner of delivery.
- Sending must be context based: `begin`, render, preview or stream, final send,
  commit, fail.
- Receiving must be context based too: normalize, dedupe, route, record,
  dispatch, platform ack, fail.
- The public plugin SDK should collapse to one small channel-message surface.

## Problems

The current channel stack grew from several valid local needs:

- Simple inbound adapters use `runtime.channel.turn.run`.
- Rich adapters use `runtime.channel.turn.runPrepared`.
- Legacy helpers use `dispatchInboundReplyWithBase`,
  `recordInboundSessionAndDispatchReply`, reply payload helpers, reply chunking,
  reply references, and outbound runtime helpers.
- Preview streaming lives in channel-specific dispatchers.
- Final delivery durability is being added around existing reply payload paths.

That shape fixes local bugs, but it leaves OpenClaw with too many public
concepts and too many places where delivery semantics can drift.

The reliability issue that exposed this is:

```text
Telegram polling update acked
  -> assistant final text exists
  -> process restarts before sendMessage succeeds
  -> final response is lost
```

The target invariant is broader than Telegram: once core decides a visible
outbound message should exist, the intent must be durable before the platform
send is attempted, and the platform receipt must be committed after success.
That gives OpenClaw at-least-once recovery. Exactly-once behavior exists only
for adapters that can prove native idempotency or reconcile an
unknown-after-send attempt against platform state before replay.

That is the end state for this refactor, not a description of every current
path. During migration, existing outbound helpers can still fall through to a
direct send when best-effort queue writes fail. The refactor is complete only
when durable final sends fail closed or explicitly opt out with a documented
non-durable policy.

## Goals

- One core lifecycle for all channel message receive and send paths.
- Durable final sends by default in the new message lifecycle after an adapter
  declares replay-safe behavior.
- Shared preview, edit, stream, finalization, retry, recovery, and receipt
  semantics.
- A small plugin SDK surface that third-party plugins can learn and maintain.
- Compatibility for existing `channel.turn` callers during migration.
- Clear extension points for new channel capabilities.
- No platform-specific branches in core.
- No token-delta channel messages. Channel streaming remains message preview,
  edit, append, or completed block delivery.
- Structured OpenClaw-origin metadata for operational/system output so visible
  gateway failures do not re-enter shared bot-enabled rooms as fresh prompts.

## Non goals

- Do not remove `runtime.channel.turn.*` in the first phase.
- Do not force every channel into the same native transport behavior.
- Do not teach core Telegram topics, Slack native streams, Matrix redactions,
  Feishu cards, QQ voice, or Teams activities.
- Do not publish all internal migration helpers as stable SDK API.
- Do not make retries replay completed non-idempotent platform operations.

## Reference model

Vercel Chat has a good public mental model:

- `Chat`
- `Thread`
- `Channel`
- `Message`
- adapter methods such as `postMessage`, `editMessage`, `deleteMessage`,
  `stream`, `startTyping`, and history fetches
- a state adapter for dedupe, locks, queues, and persistence

OpenClaw should borrow the vocabulary, not copy the surface.

What OpenClaw needs beyond that model:

- Durable outbound send intents before direct transport calls.
- Explicit send contexts with begin, commit, and fail.
- Receive contexts that know platform ack policy.
- Receipts that survive restart and can drive edits, deletes, recovery, and
  duplicate suppression.
- A smaller public SDK. Bundled plugins can use internal runtime helpers, but
  third-party plugins should see one coherent message API.
- Agent-specific behavior: sessions, transcripts, block streaming, tool
  progress, approvals, media directives, silent replies, and group mention
  history.

`thread.post()` style promises are not enough for OpenClaw. They hide the
transaction boundary that decides whether a send is recoverable.

## Core model

The new domain should live under an internal core namespace such as
`src/channels/message/*`.

It has four concepts:

```typescript
core.messages.receive(...)
core.messages.send(...)
core.messages.live(...)
core.messages.state(...)
```

`receive` owns inbound lifecycle.

`send` owns outbound lifecycle.

`live` owns preview, edit, progress, and stream state.

`state` owns durable intent storage, receipts, idempotency, recovery, locks, and
dedupe.

## Message terms

### Message

A normalized message is platform-neutral:

```typescript
type ChannelMessage = {
  id: string;
  channel: string;
  accountId?: string;
  direction: "inbound" | "outbound";
  target: MessageTarget;
  sender?: MessageActor;
  body?: MessageBody;
  attachments?: MessageAttachment[];
  relation?: MessageRelation;
  origin?: MessageOrigin;
  timestamp?: number;
  raw?: unknown;
};
```

### Target

The target describes where the message lives:

```typescript
type MessageTarget = {
  kind: "direct" | "group" | "channel" | "thread";
  id: string;
  label?: string;
  spaceId?: string;
  parentId?: string;
  threadId?: string;
  nativeChannelId?: string;
};
```

### Relation

Reply is a relation, not an API root:

```typescript
type MessageRelation =
  | {
      kind: "reply";
      inboundMessageId?: string;
      replyToId?: string;
      threadId?: string;
      quote?: MessageQuote;
    }
  | {
      kind: "followup";
      sessionKey?: string;
      previousMessageId?: string;
    }
  | {
      kind: "broadcast";
      reason?: string;
    }
  | {
      kind: "system";
      reason:
        | "approval"
        | "task"
        | "hook"
        | "cron"
        | "subagent"
        | "message_tool"
        | "cli"
        | "control_ui"
        | "automation"
        | "error";
    };
```

This lets the same send path handle normal replies, cron notifications, approval
prompts, task completions, message-tool sends, CLI or Control UI sends, subagent
results, and automation sends.

### Origin

Origin describes who produced a message and how OpenClaw should treat echoes of
that message. It is separate from relation: a message can be a reply to a user
and still be OpenClaw-originated operational output.

```typescript
type MessageOrigin =
  | {
      source: "openclaw";
      schemaVersion: 1;
      kind: "gateway_failure";
      code: "agent_failed_before_reply" | "missing_api_key" | "model_login_expired";
      echoPolicy: "drop_bot_room_echo";
    }
  | {
      source: "user" | "external_bot" | "platform" | "unknown";
    };
```

Core owns the meaning of OpenClaw-originated output. Channels own how that
origin is encoded into their transport.

The first required use is gateway failure output. Humans should still see
messages such as "Agent failed before reply" or "Missing API key", but tagged
OpenClaw operational output must not be accepted as bot-authored input in shared
rooms when `allowBots` is enabled.

### Receipt

Receipts are first-class:

```typescript
type MessageReceipt = {
  primaryPlatformMessageId?: string;
  platformMessageIds: string[];
  parts: MessageReceiptPart[];
  threadId?: string;
  replyToId?: string;
  editToken?: string;
  deleteToken?: string;
  url?: string;
  sentAt: number;
  raw?: unknown;
};

type MessageReceiptPart = {
  platformMessageId: string;
  kind: "text" | "media" | "voice" | "card" | "preview" | "unknown";
  index: number;
  threadId?: string;
  replyToId?: string;
  editToken?: string;
  deleteToken?: string;
  url?: string;
  raw?: unknown;
};
```

Receipts are the bridge from durable intent to future edit, delete, preview
finalization, duplicate suppression, and recovery.

A receipt can describe one platform message or a multi-part delivery. Chunked
text, media plus text, voice plus text, and card fallbacks must preserve all
platform ids while still exposing a primary id for threading and later edits.

## Receive context

Receiving should not be a bare helper call. The core needs a context that knows
dedupe, routing, session recording, and platform ack policy.

```typescript
type MessageReceiveContext = {
  id: string;
  channel: string;
  accountId?: string;
  input: ChannelMessage;
  ack: ReceiveAckController;
  route: MessageRouteController;
  session: MessageSessionController;
  log: MessageLifecycleLogger;

  dedupe(): Promise;
  resolve(): Promise;
  record(resolved: ResolvedInboundMessage): Promise;
  dispatch(recorded: RecordResult): Promise;
  commit(result: DispatchResult): Promise<void>;
  fail(error: unknown): Promise<void>;
};
```

Receive flow:

```text
platform event
  -> begin receive context
  -> normalize
  -> classify
  -> dedupe and self-echo gate
  -> route and authorize
  -> record inbound session metadata
  -> dispatch agent run
  -> durable outbound sends happen through send context
  -> commit receive
  -> ack platform when policy allows
```

Ack is not one thing. The receive contract must keep these signals separate:

- **Transport ack:** tells the platform webhook or socket that OpenClaw accepted
  the event envelope. Some platforms require this before dispatch.
- **Polling offset ack:** advances a cursor so the same event is not fetched
  again. This must not advance past work that cannot be recovered.
- **Inbound record ack:** confirms OpenClaw persisted enough inbound metadata to
  dedupe and route a redelivery.
- **User-visible receipt:** optional read/status/typing behavior; never a
  durability boundary.

`ReceiveAckPolicy` controls transport or polling acknowledgement only. It must
not be reused for read receipts or status reactions.

Before bot authorization, receive must apply the shared OpenClaw echo policy
when the channel can decode message origin metadata:

```typescript
function shouldDropOpenClawEcho(params: {
  origin?: MessageOrigin;
  isBotAuthor: boolean;
  isRoomish: boolean;
}): boolean {
  return (
    params.isBotAuthor &&
    params.isRoomish &&
    params.origin?.source === "openclaw" &&
    params.origin.kind === "gateway_failure" &&
    params.origin.echoPolicy === "drop_bot_room_echo"
  );
}
```

This drop is tag-based, not text-based. A bot-authored room message with the
same visible gateway-failure text but without OpenClaw origin metadata still
goes through normal `allowBots` authorization.

Ack policy is explicit:

```typescript
type ReceiveAckPolicy =
  | { kind: "immediate"; reason: "webhook-timeout" | "platform-contract" }
  | { kind: "after-record" }
  | { kind: "after-durable-send" }
  | { kind: "manual" };
```

Telegram polling now uses the receive-context ack policy for its persisted
restart watermark. The tracker still observes grammY updates as they enter the
middleware chain, but OpenClaw persists only the safe completed update id after
successful dispatch, leaving failed or lower pending updates replayable after a
restart. Telegram's upstream `getUpdates` fetch offset is still controlled by
the polling library, so the remaining deeper cut is a fully durable polling
source if we need platform-level redelivery beyond OpenClaw's restart
watermark. Webhook platforms may need immediate HTTP ack, but they still need
inbound dedupe and durable outbound send intents because webhooks can redeliver.

## Send context

Sending is also context based:

```typescript
type MessageSendContext = {
  id: string;
  channel: string;
  accountId?: string;
  message: ChannelMessage;
  intent: DurableSendIntent;
  attempt: number;
  signal: AbortSignal;
  previousReceipt?: MessageReceipt;
  preview?: LiveMessageState;
  log: MessageLifecycleLogger;

  render(): Promise;
  previewUpdate(rendered: RenderedMessageBatch): Promise;
  send(rendered: RenderedMessageBatch): Promise;
  edit(receipt: MessageReceipt, rendered: RenderedMessageBatch): Promise;
  delete(receipt: MessageReceipt): Promise<void>;
  commit(receipt: MessageReceipt): Promise<void>;
  fail(error: unknown): Promise<void>;
};
```

Preferred orchestration:

```typescript
await core.messages.withSendContext(message, async (ctx) => {
  const rendered = await ctx.render();

  if (ctx.preview?.canFinalizeInPlace) {
    return await ctx.edit(ctx.preview.receipt, rendered);
  }

  return await ctx.send(rendered);
});
```

The helper expands to:

```text
begin durable intent
  -> render
  -> optional preview/edit/stream work
  -> mark sending
  -> final platform send or final edit
  -> mark committing with raw receipt
  -> commit receipt
  -> ack durable intent
  -> fail durable intent on classified failure
```

The intent must exist before transport I/O. A restart after begin but before
commit is recoverable.

The dangerous boundary is after platform success and before receipt commit. If a
process dies there, OpenClaw cannot know whether the platform message exists
unless the adapter provides native idempotency or a receipt reconciliation path.
Those attempts must resume in `unknown_after_send`, not blindly replay. Channels
without reconciliation may choose at-least-once replay only if duplicate visible
messages are an acceptable, documented tradeoff for that channel and relation.
The current SDK reconciliation bridge requires the adapter to declare
`reconcileUnknownSend`, then asks `durableFinal.reconcileUnknownSend` to
classify an unknown entry as `sent`, `not_sent`, or `unresolved`; only `not_sent`
permits replay, and unresolved entries stay terminal or retry only the
reconciliation che
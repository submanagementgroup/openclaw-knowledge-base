---
domain: plugins
topic: "Channel Message API"
type: reference
keywords:
  - channel message
  - message API
  - SDK message
  - message format
  - outbound messages
source: plugins/sdk-channel-message.md
---

Channel plugins should expose one `message` adapter from
`openclaw/plugin-sdk/channel-message`. The adapter describes the native message
lifecycle that the platform supports:

```text
receive -> route and record -> agent turn -> durable final send
send -> render batch -> platform I/O -> receipt -> lifecycle side effects
live preview -> final edit or fallback -> receipt
```

Core owns queueing, durability, generic retry policy, hooks, receipts, and the
shared `message` tool. The plugin owns native send/edit/delete calls, target
normalization, platform threading, selected quotes, notification flags, account
state, and platform-specific side effects.

Use this page together with [Building channel plugins](/plugins/sdk-channel-plugins).

The `channel-message` subpath is intentionally cheap enough for hot plugin
bootstrap files such as `channel.ts`: it exposes adapter contracts, capability
proofs, receipts, and compatibility facades without loading outbound delivery.
Runtime delivery helpers are available from
`openclaw/plugin-sdk/channel-message-runtime` for monitor/send code paths that
are already doing asynchronous message I/O.

New channel and plugin send code should use the message lifecycle helpers from
`openclaw/plugin-sdk/channel-message-runtime`: `sendDurableMessageBatch`,
`withDurableMessageSendContext`, or `deliverInboundReplyWithMessageSendContext`.
The older
`deliverOutboundPayloads(...)` helper in `openclaw/plugin-sdk/outbound-runtime`
is deprecated compatibility/runtime substrate for outbound internals, recovery,
and legacy adapters. Do not use it for new channel or plugin send paths.

`sendDurableMessageBatch(...)` returns an explicit lifecycle outcome:

- `sent` - at least one visible platform message was delivered.
- `suppressed` - no platform message should be treated as missing. Stable
  reasons include `cancelled_by_message_sending_hook`,
  `empty_after_message_sending_hook`, `no_visible_payload`,
  `adapter_returned_no_identity`, and legacy `no_visible_result`.
- `partial_failed` - at least one platform message was delivered before a later
  payload or side effect failed. The result includes the delivered receipt prefix
  plus the failure.
- `failed` - no platform receipt was produced.

Use `payloadOutcomes` when a batch mixes sent, suppressed, and failed payloads.
Do not infer hook cancellation by checking whether the old direct-delivery array
is empty.

Compatibility dispatchers that still need the buffered reply dispatcher should
build reply-prefix options with `createChannelMessageReplyPipeline(...)` from
`openclaw/plugin-sdk/channel-message`, then call the runtime's
`channel.turn.runPrepared(...)`. That keeps session recording and dispatch
ordering on the shared turn lifecycle without adding another public turn wrapper.

## Minimal adapter

Most new channel plugins can start with a small adapter:

```typescript
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-message";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

Then attach it to the channel plugin:

```typescript
export const demoPlugin = createChatChannelPlugin({
  base: {
    id: "demo",
    message: demoMessageAdapter,
    // other channel plugin fields
  },
});
```

Only declare capabilities that the adapter really preserves. Every declared
capability should have a contract test.

## Outbound bridge

If the channel already has a compatible `outbound` adapter, prefer deriving the
message adapter instead of duplicating send code:

```typescript
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-message";

const demoMessageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound: demoOutboundAdapter,
});
```

The bridge converts old outbound send results into `MessageReceipt` values. New
code should pass receipts end to end and only derive legacy ids at compatibility
edges with `listMessageReceiptPlatformIds(...)` or
`resolveMessageReceiptPrimaryId(...)`.
If no receive policy is supplied, `createChannelMessageAdapterFromOutbound(...)`
uses `manual` receive acknowledgement policy. That makes plugin-owned platform
acknowledgement explicit without changing channels that acknowledge webhooks,
sockets, or polling offsets outside generic receive context.

## Message tool sends

The shared `message(action="send")` path should use the same core delivery
lifecycle as final replies. If a channel needs provider-specific shaping for the
tool send, implement `actions.prepareSendPayload(...)` instead of sending from
`actions.handleAction(...)`.

`prepareSendPayload(...)` receives the normalized core `ReplyPayload` plus the
full action context. Return a payload with channel-specific data in
`payload.channelData.<channel>` and let core call `sendMessage(...)`,
the message lifecycle runtime, the write-ahead queue, message-sending hooks,
retry, recovery, and ack cleanup. The lifecycle runtime may call
`deliverOutboundPayloads(...)` internally as compatibility substrate, but channel
plugins should not call it directly for new send behavior.

Return `null` only when the send cannot be represented as a durable payload, for
example because it contains a non-serializable component factory. Core will keep
the legacy plugin action fallback for compatibility, but new channel send
features should be expressible as durable payload data.

```typescript
export const demoActions: ChannelMessageActionAdapter = {
  describeMessageTool: () => ({ actions: ["send"], capabilities: ["presentation"] }),
  prepareSendPayload: ({ ctx, payload }) => {
    if (ctx.action !== "send") {
      return null;
    }
    return {
      ...payload,
      channelData: {
        ...payload.channelData,
        demo: {
          ...(payload.channelData?.demo as object | undefined),
          nativeCard: ctx.params.card,
        },
      },
    };
  },
};
```

The outbound adapter then reads `payload.channelData.demo` inside `sendPayload`.
This keeps platform-specific rendering in the plugin while core still owns
persist, retry, recover, hooks, and ack.

Prepared `message(action="send")` payloads and generic final-reply delivery use
core delivery with best-effort queueing by default. Required durable queueing is
only valid after core verifies the channel can reconcile a send whose outcome is
unknown after a crash. If the adapter cannot implement `reconcileUnknownSend`,
keep the prepared send path best-effort; core will still try the write-ahead
queue, but queue persistence or uncertain crash recovery is not part of the
required delivery contract.

## Durable final capabilities

Durable final delivery is opt in per side effect. Core will only use generic
durable delivery when the adapter declares every capability needed by the
payload and delivery options.

| Capability             | Declare when                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------ |
| `text`                 | The adapter can send text and return a receipt.                                      |
| `media`                | Media sends return receipts for every visible platform message.                      |
| `payload`              | The adapter preserves rich reply payload semantics, not only text and one media URL. |
| `replyTo`              | Native reply targets reach the platform.                                             |
| `thread`               | Native thread, topic, or channel thread targets reach the platform.                  |
| `silent`               | Notification suppression reaches the platform.                                       |
| `nativeQuote`          | Selected quote metadata reaches the platform.                                        |
| `messageSendingHooks`  | Core message-sending hooks can cancel or rewrite content before platform I/O.        |
| `batch`                | Multi-part rendered batches are replayable as one durable plan.                      |
| `reconcileUnknownSend` | The adapter can resolve `unknown_after_send` recovery without blind replay.          |
| `afterSendSuccess`     | Channel-local after-send side effects run once.                                      |
| `afterCommit`          | Channel-local after-commit side effects run once.                                    |

Best-effort final delivery does not require `reconcileUnknownSend`; it uses the
shared lifecycle when the adapter preserves the payload's visible semantics, and
falls back to direct platform I/O if queue persistence is unavailable. Required
durable final delivery must explicitly require `reconcileUnknownSend`. If the
adapter cannot determine whether a started/unknown send reached the platform,
do not declare that capability; core will reject required durable delivery
before queueing.

When a caller needs durable delivery, derive requirements instead of building
maps by hand:

```typescript
import { deriveDurableFinalDeliveryRequirements } from "openclaw/plugin-sdk/channel-message";

const requiredCapabilities = deriveDurableFinalDeliveryRequirements({
  payload,
  replyToId,
  threadId,
  silent,
  payloadTransport: true,
  extraCapabilities: {
    nativeQuote: hasSelectedQuote(payload),
  },
});
```

`messageSendingHooks` is required by default. Set `messageSendingHooks: false`
only for a path that intentionally cannot run global message-sending hooks.

## Durable send contract

A durable final send has stricter semantics than legacy channel-owned delivery:

- Create the durable intent before platform I/O.
- If durable delivery returns a handled result, do not fall back to legacy send.
- Treat hook cancellation and no-send results as terminal.
- Treat `unsupported` as a pre-intent result only.
- For required durability, fail before platform I/O if the queue cannot record
  that platform send has started.
- For required final delivery and required prepared message-tool sends,
  preflight `reconcileUnknownSend`; recovery must be able to ack an
  already-sent message or replay only after the adapter proves the original send
  did not happen.
- For `best_effort`, queue write failures may fall back to direct platform I/O.
- Forward abort signals to media loading and platform sends.
- Run after-commit hooks after queue ack; direct best-effort fallback runs them
  after successful platform I/O because there is no durable queue commit.
- Return receipts for every visible platform message id.
- Use `reconcileUnknownSend` when a platform can check whether an uncertain send
  already reached the user.

This contract avoids duplicate sends after crashes and avoids bypassing
message-sending cancellation hooks.

## Receipts

`MessageReceipt` is the new internal record of what the platform accepted:

```typescript
type MessageReceipt = {
  primaryPlatformMessageId?: string;
  platformMessageIds: string[];
  parts: MessageReceiptPart[];
  threadId?: string;
  replyToId?: string;
  editToken?: string;
  deleteToken?: string;
  sentAt: number;
  raw?: readonly MessageReceiptSourc
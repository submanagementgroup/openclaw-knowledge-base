---
domain: plugins
topic: "SDK Channel Plugins: Building Custom Messaging Channel Integrations"
type: procedure
keywords:
  - channel plugins
  - SDK channel
  - channel implementation
  - channel bridge
  - messaging plugin
related:
  - plugins/sdk-overview
  - plugins/sdk-channel-turn
source: plugins/sdk-channel-plugins.md
---

Building channel plugins with the OpenClaw SDK. Channels bridge external messaging platforms to the OpenClaw Gateway.

This guide walks through building a channel plugin that connects OpenClaw to a
messaging platform. By the end you will have a working channel with DM security,
pairing, reply threading, and outbound messaging.

  If you have not built any OpenClaw plugin before, read
  [Getting Started](/plugins/building-plugins) first for the basic package
  structure and manifest setup.

## How channel plugins work

Channel plugins do not need their own send/edit/react tools. OpenClaw keeps one
shared `message` tool in core. Your plugin owns:

- **Config** — account resolution and setup wizard
- **Security** — DM policy and allowlists
- **Pairing** — DM approval flow
- **Session grammar** — how provider-specific conversation ids map to base chats, thread ids, and parent fallbacks
- **Outbound** — sending text, media, and polls to the platform
- **Threading** — how replies are threaded
- **Heartbeat typing** — optional typing/busy signals for heartbeat delivery targets

Core owns the shared message tool, prompt wiring, the outer session-key shape,
generic `:thread:` bookkeeping, and dispatch.

If your channel supports typing indicators outside inbound replies, expose
`heartbeat.sendTyping(...)` on the channel plugin. Core calls it with the
resolved heartbeat delivery target before the heartbeat model run starts and
uses the shared typing keepalive/cleanup lifecycle. Add `heartbeat.clearTyping(...)`
when the platform needs an explicit stop signal.

If your channel adds message-tool params that carry media sources, expose those
param names through `describeMessageTool(...).mediaSourceParams`. Core uses
that explicit list for sandbox path normalization and outbound media-access
policy, so plugins do not need shared-core special cases for provider-specific
avatar, attachment, or cover-image params.
Prefer returning an action-keyed map such as
`{ "set-profile": ["avatarUrl", "avatarPath"] }` so unrelated actions do not
inherit another action's media args. A flat array still works for params that
are intentionally shared across every exposed action.

If your platform stores extra scope inside conversation ids, keep that parsing
in the plugin with `messaging.resolveSessionConversation(...)`. That is the
canonical hook for mapping `rawId` to the base conversation id, optional thread
id, explicit `baseConversationId`, and any `parentConversationCandidates`.
When you return `parentConversationCandidates`, keep them ordered from the
narrowest parent to the broadest/base conversation.

Use `openclaw/plugin-sdk/channel-route` when plugin code needs to normalize
route-like fields, compare a child thread with its parent route, or build a
stable dedupe key from `{ channel, to, accountId, threadId }`. The helper
normalizes numeric thread ids the same way core does, so plugins should prefer
it over ad hoc `String(threadId)` comparisons.
Plugins with provider-specific target grammar can inject their parser into
`resolveChannelRouteTargetWithParser(...)` and still get the same route target
shape and thread fallback semantics core uses.

Bundled plugins that need the same parsing before the channel registry boots
can also expose a top-level `session-key-api.ts` file with a matching
`resolveSessionConversation(...)` export. Core uses that bootstrap-safe surface
only when the runtime plugin registry is not available yet.

`messaging.resolveParentConversationCandidates(...)` remains available as a
legacy compatibility fallback when a plugin only needs parent fallbacks on top
of the generic/raw id. If both hooks exist, core uses
`resolveSessionConversation(...).parentConversationCandidates` first and only
falls back to `resolveParentConversationCandidates(...)` when the canonical hook
omits them.

## Approvals and channel capabilities

Most channel plugins do not need approval-specific code.

- Core owns same-chat `/approve`, shared approval button payloads, and generic fallback delivery.
- Prefer one `approvalCapability` object on the channel plugin when the channel needs approval-specific behavior.
- `ChannelPlugin.approvals` is removed. Put approval delivery/native/render/auth facts on `approvalCapability`.
- `plugin.auth` is login/logout only; core no longer reads approval auth hooks from that object.
- `approvalCapability.authorizeActorAction` and `approvalCapability.getActionAvailabilityState` are the canonical approval-auth seam.
- Use `approvalCapability.getActionAvailabilityState` for same-chat approval auth availability.
- If your channel exposes native exec approvals, use `approvalCapability.getExecInitiatingSurfaceState` for the initiating-surface/native-client state when it differs from same-chat approval auth. Core uses that exec-specific hook to distinguish `enabled` vs `disabled`, decide whether the initiating channel supports native exec approvals, and include the channel in native-client fallback guidance. `createApproverRestrictedNativeApprovalCapability(...)` fills this in for the common case.
- Use `outbound.shouldSuppressLocalPayloadPrompt` or `outbound.beforeDeliverPayload` for channel-specific payload lifecycle behavior such as hiding duplicate local approval prompts or sending typing indicators before delivery.
- Use `approvalCapability.delivery` only for native approval routing or fallback suppression.
- Use `approvalCapability.nativeRuntime` for channel-owned native approval facts. Keep it lazy on hot channel entrypoints with `createLazyChannelApprovalNativeRuntimeAdapter(...)`, which can import your runtime module on demand while still letting core assemble the approval lifecycle.
- Use `approvalCapability.render` only when a channel truly needs custom approval payloads instead of the shared renderer.
- Use `approvalCapability.describeExecApprovalSetup` when the channel wants the disabled-path reply to explain the exact config knobs needed to enable native exec approvals. The hook receives `{ channel, channelLabel, accountId }`; named-account channels should render account-scoped paths such as `channels.<channel>.accounts.<id>.execApprovals.*` instead of top-level defaults.
- If a channel can infer stable owner-like DM identities from existing config, use `createResolvedApproverActionAuthAdapter` from `openclaw/plugin-sdk/approval-runtime` to restrict same-chat `/approve` without adding approval-specific core logic.
- If a channel needs native approval delivery, keep channel code focused on target normalization plus transport/presentation facts. Use `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, and `createApproverRestrictedNativeApprovalCapability` from `openclaw/plugin-sdk/approval-runtime`. Put the channel-specific facts behind `approvalCapability.nativeRuntime`, ideally via `createChannelApprovalNativeRuntimeAdapter(...)` or `createLazyChannelApprovalNativeRuntimeAdapter(...)`, so core can assemble the handler and own request filtering, routing, dedupe, expiry, gateway subscription, and routed-elsewhere notices. `nativeRuntime` is split into a few smaller seams:
- `createChannelNativeOriginTargetResolver` uses the shared channel-route matcher by default for `{ to, accountId, threadId }` targets. Pass `targetsMatch` only when a channel has provider-specific equivalence rules, such as Slack timestamp prefix matching.
- Pass `normalizeTargetForMatch` to `createChannelNativeOriginTargetResolver` when the channel needs to canonicalize provider ids before the default route matcher or a custom `targetsMatch` callback runs, while preserving the original target for delivery. Use `normalizeTarget` only when the resolved delivery target itself should be canonicalized.
- `availability` — whether the account is configured and whether a request should be handled
- `presentation` — map the shared approval view model into pending/resolved/expired native payloads or final actions
- `transport` — prepare targets plus send/update/delete native approval messages
- `interactions` — optional bind/unbind/clear-action hooks for native buttons or reactions
- `observe` — optional delivery diagnostics hooks
- If the channel needs runtime-owned objects such as a client, token, Bolt app, or webhook receiver, register them through `openclaw/plugin-sdk/channel-runtime-context`. The generic runtime-context registry lets core bootstrap capability-driven handlers from channel startup state without adding approval-specific wrapper glue.
- Reach for the lower-level `createChannelApprovalHandler` or `createChannelNativeApprovalRuntime` only when the capability-driven seam is not expressive enough yet.
- Native approval channels must route both `accountId` and `approvalKind` through those helpers. `accountId` keeps multi-account approval policy scoped to the right bot account, and `approvalKind` keeps exec vs plugin approval behavior available to the channel without hardcoded branches in core.
- Core now owns approval reroute notices too. Channel plugins should not send their own "approval went to DMs /

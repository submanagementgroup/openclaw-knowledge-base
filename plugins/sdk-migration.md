---
domain: plugins
topic: "Plugin SDK Migration Guide"
type: procedure
keywords:
  - SDK migration
  - plugin migration
  - upgrade plugins
  - breaking changes
  - migration guide
source: plugins/sdk-migration.md
---

OpenClaw has moved from a broad backwards-compatibility layer to a modern plugin
architecture with focused, documented imports. If your plugin was built before
the new architecture, this guide helps you migrate.

## What is changing

The old plugin system provided two wide-open surfaces that let plugins import
anything they needed from a single entry point:

- **`openclaw/plugin-sdk/compat`** - a single import that re-exported dozens of
  helpers. It was introduced to keep older hook-based plugins working while the
  new plugin architecture was being built.
- **`openclaw/plugin-sdk/infra-runtime`** - a broad runtime helper barrel that
  mixed system events, heartbeat state, delivery queues, fetch/proxy helpers,
  file helpers, approval types, and unrelated utilities.
- **`openclaw/plugin-sdk/config-runtime`** - a broad config compatibility barrel
  that still carries deprecated direct load/write helpers during the migration
  window.
- **`openclaw/extension-api`** - a bridge that gave plugins direct access to
  host-side helpers like the embedded agent runner.
- **`api.registerEmbeddedExtensionFactory(...)`** - a removed Pi-only bundled
  extension hook that could observe embedded-runner events such as
  `tool_result`.

The broad import surfaces are now **deprecated**. They still work at runtime,
but new plugins must not use them, and existing plugins should migrate before
the next major release removes them. The Pi-only embedded extension factory
registration API has been removed; use tool-result middleware instead.

OpenClaw does not remove or reinterpret documented plugin behavior in the same
change that introduces a replacement. Breaking contract changes must first go
through a compatibility adapter, diagnostics, docs, and a deprecation window.
That applies to SDK imports, manifest fields, setup APIs, hooks, and runtime
registration behavior.

> **Note:** The backwards-compatibility layer will be removed in a future major release.
  Plugins that still import from these surfaces will break when that happens.
  Pi-only embedded extension factory registrations already no longer load.


## Why this changed

The old approach caused problems:

- **Slow startup** - importing one helper loaded dozens of unrelated modules
- **Circular dependencies** - broad re-exports made it easy to create import cycles
- **Unclear API surface** - no way to tell which exports were stable vs internal

The modern plugin SDK fixes this: each import path (`openclaw/plugin-sdk/\<subpath\>`)
is a small, self-contained module with a clear purpose and documented contract.

Legacy provider convenience seams for bundled channels are also gone.
Channel-branded helper seams were private mono-repo shortcuts, not stable
plugin contracts. Use narrow generic SDK subpaths instead. Inside the bundled
plugin workspace, keep provider-owned helpers in that plugin's own `api.ts` or
`runtime-api.ts`.

Current bundled provider examples:

- Anthropic keeps Claude-specific stream helpers in its own `api.ts` /
  `contract-api.ts` seam
- OpenAI keeps provider builders, default-model helpers, and realtime provider
  builders in its own `api.ts`
- OpenRouter keeps provider builder and onboarding/config helpers in its own
  `api.ts`

## Talk and realtime voice migration plan

Realtime voice, telephony, meeting, and browser Talk code is moving from
surface-local turn bookkeeping to a shared Talk session controller exported by
`openclaw/plugin-sdk/realtime-voice`. The new controller owns the common Talk
event envelope, active turn state, capture state, output-audio state, recent
event history, and stale-turn rejection. Provider plugins should keep owning
vendor-specific realtime sessions; surface plugins should keep owning capture,
playback, telephony, and meeting quirks.

This Talk migration is intentionally breaking-clean:

1. Keep the shared controller/runtime primitives in
   `plugin-sdk/realtime-voice`.
2. Move bundled surfaces onto the shared controller: browser relay,
   managed-room handoff, voice-call realtime, voice-call streaming STT, Google
   Meet realtime, and native push-to-talk.
3. Replace old Talk RPC families with the final `talk.session.*` and
   `talk.client.*` API.
4. Advertise one live Talk event channel in Gateway
   `hello-ok.features.events`: `talk.event`.
5. Delete the old realtime HTTP endpoint and any request-time instruction
   override path.

New code should not call `createTalkEventSequencer(...)` directly unless it is
implementing a low-level adapter or test fixture. Prefer the shared controller
so turn-scoped events cannot be emitted without a turn id, stale `turnEnd` /
`turnCancel` calls cannot clear a newer active turn, and output-audio lifecycle
events stay consistent across telephony, meetings, browser relay, managed-room
handoff, and native Talk clients.

The target public API shape is:

```typescript
// Gateway-owned Talk session API.
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// Client-owned provider session API.
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
```

Browser-owned WebRTC/provider-websocket sessions use `talk.client.create`,
because the browser owns the provider negotiation and media transport while the
Gateway owns credentials, instructions, and tool policy. `talk.session.*` is the
common Gateway-managed surface for gateway-relay realtime, gateway-relay
transcription, and managed-room native STT/TTS sessions.

Legacy configs that placed realtime selectors beside `talk.provider` /
`talk.providers` should be repaired with `openclaw doctor --fix`; runtime Talk
does not reinterpret speech/TTS provider config as realtime provider config.

The supported `talk.session.create` combinations are intentionally small:

| Mode            | Transport       | Brain           | Owner              | Notes                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | Full-duplex provider audio bridged through the Gateway; tool calls are routed through the agent-consult tool.      |
| `transcription` | `gateway-relay` | `none`          | Gateway            | Streaming STT only; callers send input audio and receive transcript events.                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | Native/client room | Push-to-talk and walkie-talkie style rooms where the client owns capture/playback and the Gateway owns turn state. |
| `stt-tts`       | `managed-room`  | `direct-tools`  | Native/client room | Admin-only room mode for trusted first-party surfaces that execute Gateway tool actions directly.                  |

Removed method map:

| Old                              | New                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` or `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

The unified control vocabulary is also deliberately narrow:

| Method                          | Applies to                                              | Contract                                                                                                                                                                                 |
| ------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`, `transcription/gateway-relay` | Append a base64 PCM audio chunk to the provider session owned by the same Gateway connection.                                                                                            |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | Start a managed-room user turn.                                                                                                                                                          |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | End the active turn after stale-turn validation.                                                                                                                                         |
| `talk.session.cancelTurn`       | all Gateway-owned sessions                              | Cancel active capture/provider/agent/TTS work for a turn.                                                                                                                                |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | Stop assistant audio output without necessarily ending the user turn.                                                                                                                    |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | Complete a provider tool call emitted by the relay; pass `options.willContinue` for interim output or `options.suppressResponse` to satisfy the call without another assistant response. |
| `talk.session.close`            | all unified sessions                                    | Stop relay sessions or revoke managed-room state, then forget the unified session id.                                                                                                    |

Do not introduce provider or platform special cases in core to make this work.
Core owns Talk session semantics. Provider plugins own vendor session setup.
Voice-call and Google Meet own telephony/meeting adapters. Browser and native
apps own device capture/playback UX.

## Compatibility policy

For external plugins, compatibility work follows this order:

1. add the new contract
2. keep the old behavior wired through a compatibility adapter
3. emit a diagnostic or warning that names the old path and replacement
4. cover both paths in tests
5. document the deprecation and migration path
6. remove only after the announced migration window, usually in a major release

Maintainers can audit the current migration queue with
`pnpm plugins:boundary-report`. Use `pnpm plugins:boundary-report:summary` for
compact counts, `--owner <id>` for one plugin or compatibility owner, and
`pnpm plugins:boundary-report:ci` when a CI gate should fail on due
compatibility records, cross-owner reserved SDK imports, or unused reserved SDK
subpaths. The report groups deprecated
compatibility records by removal date, counts local code/docs references,
surfaces cross-owner reserved SDK imports, and summarizes the private
memory-host SDK bridge so compatibility cleanup stays explicit instead of
relying on ad hoc searches. Reserved SDK subpaths must have tracked owner usage;
unused reserved helper exports should be removed from the public SDK.

If a manifest field is still accepted, plugin authors can keep using it until
the docs and diagnostics say otherwise. New code should prefer the documented
replacement, but existing plugins should not break during ordinary minor
releases.

## How to migrate

**Migrate runtime config load/write helpers**

Bundled plugins should stop calling
    `api.runtime.config.loadConfig()` and
    `api.runtime.config.writeConfigFile(...)` directly. Prefer config that was
    already passed into the active call path. Long-lived handlers that need the
    current process snapshot can use `api.runtime.config.current()`. Long-lived
    agent tools should use the tool context's `ctx.getRuntimeConfig()` inside
    `execut
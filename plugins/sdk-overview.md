---
domain: plugins
topic: "Plugin SDK Overview"
type: concept
keywords:
  - plugin SDK
  - SDK
  - plugin development
  - openclaw SDK
  - extension
  - plugin architecture
source: plugins/sdk-overview.md
---

The plugin SDK is the typed contract between plugins and core. This page is the
reference for **what to import** and **what you can register**.

> **Note:** This page is for plugin authors using `openclaw/plugin-sdk/*` inside
  OpenClaw. For external apps, scripts, dashboards, CI jobs, and IDE extensions
  that want to run agents through the Gateway, use the
  [OpenClaw App SDK](/concepts/openclaw-sdk) and the `@openclaw/sdk` package
  instead.


> **Note:** Looking for a how-to guide instead? Start with [Building plugins](/plugins/building-plugins), use [Channel plugins](/plugins/sdk-channel-plugins) for channel plugins, [Provider plugins](/plugins/sdk-provider-plugins) for provider plugins, [CLI backend plugins](/plugins/cli-backend-plugins) for local AI CLI backends, and [Plugin hooks](/plugins/hooks) for tool or lifecycle hook plugins.


## Import convention

Always import from a specific subpath:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Each subpath is a small, self-contained module. This keeps startup fast and
prevents circular dependency issues. For channel-specific entry/build helpers,
prefer `openclaw/plugin-sdk/channel-core`; keep `openclaw/plugin-sdk/core` for
the broader umbrella surface and shared helpers such as
`buildChannelConfigSchema`.

For channel config, publish the channel-owned JSON Schema through
`openclaw.plugin.json#channelConfigs`. The `plugin-sdk/channel-config-schema`
subpath is for shared schema primitives and the generic builder. OpenClaw's
bundled plugins use `plugin-sdk/bundled-channel-config-schema` for retained
bundled-channel schemas. Deprecated compatibility exports remain on
`plugin-sdk/channel-config-schema-legacy`; neither bundled schema subpath is a
pattern for new plugins.

> **Note:** Do not import provider- or channel-branded convenience seams (for example
  `openclaw/plugin-sdk/slack`, `.../discord`, `.../signal`, `.../whatsapp`).
  Bundled plugins compose generic SDK subpaths inside their own `api.ts` /
  `runtime-api.ts` barrels; core consumers should either use those plugin-local
  barrels or add a narrow generic SDK contract when a need is truly
  cross-channel.

A small set of bundled-plugin helper seams still appear in the generated export
map when they have tracked owner usage. They exist for bundled-plugin
maintenance only and are not recommended import paths for new third-party
plugins.

`openclaw/plugin-sdk/discord` and `openclaw/plugin-sdk/telegram-account` are
also kept as deprecated compatibility facades for tracked owner usage. Do not
copy those import paths into new plugins; use injected runtime helpers and
generic channel SDK subpaths instead.


## Subpath reference

The plugin SDK is exposed as a set of narrow subpaths grouped by area (plugin
entry, channel, provider, auth, runtime, capability, memory, and reserved
bundled-plugin helpers). For the full catalog — grouped and linked — see
[Plugin SDK subpaths](/plugins/sdk-subpaths).

The compiler entrypoint inventory lives in
`scripts/lib/plugin-sdk-entrypoints.json`; package exports are generated from
the public subset after subtracting repo-local test/internal subpaths listed in
`scripts/lib/plugin-sdk-private-local-only-subpaths.json`. Run
`pnpm plugin-sdk:surface` to audit the public export count. Deprecated public
subpaths that are old enough and unused by bundled extension production code are
tracked in `scripts/lib/plugin-sdk-deprecated-public-subpaths.json`; broad
deprecated re-export barrels are tracked in
`scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json`.

## Registration API

The `register(api)` callback receives an `OpenClawPluginApi` object with these
methods:

### Capability registration

| Method                                           | What it registers                     |
| ------------------------------------------------ | ------------------------------------- |
| `api.registerProvider(...)`                      | Text inference (LLM)                  |
| `api.registerAgentHarness(...)`                  | Experimental low-level agent executor |
| `api.registerCliBackend(...)`                    | Local CLI inference backend           |
| `api.registerChannel(...)`                       | Messaging channel                     |
| `api.registerSpeechProvider(...)`                | Text-to-speech / STT synthesis        |
| `api.registerRealtimeTranscriptionProvider(...)` | Streaming realtime transcription      |
| `api.registerRealtimeVoiceProvider(...)`         | Duplex realtime voice sessions        |
| `api.registerMediaUnderstandingProvider(...)`    | Image/audio/video analysis            |
| `api.registerImageGenerationProvider(...)`       | Image generation                      |
| `api.registerMusicGenerationProvider(...)`       | Music generation                      |
| `api.registerVideoGenerationProvider(...)`       | Video generation                      |
| `api.registerWebFetchProvider(...)`              | Web fetch / scrape provider           |
| `api.registerWebSearchProvider(...)`             | Web search                            |

### Tools and commands

| Method                          | What it registers                             |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Agent tool (required or `{ optional: true }`) |
| `api.registerCommand(def)`      | Custom command (bypasses the LLM)             |

Plugin commands can set `agentPromptGuidance` when the agent needs a short,
command-owned routing hint. Keep that text about the command itself; do not add
provider- or plugin-specific policy to core prompt builders.

### Infrastructure

| Method                                         | What it registers                       |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Event hook                              |
| `api.registerHttpRoute(params)`                | Gateway HTTP endpoint                   |
| `api.registerGatewayMethod(name, handler)`     | Gateway RPC method                      |
| `api.registerGatewayDiscoveryService(service)` | Local Gateway discovery advertiser      |
| `api.registerCli(registrar, opts?)`            | CLI subcommand                          |
| `api.registerNodeCliFeature(registrar, opts?)` | Node feature CLI under `openclaw nodes` |
| `api.registerService(service)`                 | Background service                      |
| `api.registerInteractiveHandler(registration)` | Interactive handler                     |
| `api.registerAgentToolResultMiddleware(...)`   | Runtime tool-result middleware          |
| `api.registerMemoryPromptSupplement(builder)`  | Additive memory-adjacent prompt section |
| `api.registerMemoryCorpusSupplement(adapter)`  | Additive memory search/read corpus      |

### Host hooks for workflow plugins

Host hooks are the SDK seams for plugins that need to participate in the host
lifecycle rather than only adding a provider, channel, or tool. They are
generic contracts; Plan Mode can use them, but so can approval workflows,
workspace policy gates, background monitors, setup wizards, and UI companion
plugins.

| Method                                                                               | Contract it owns                                                                                                                  |
| ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | Plugin-owned, JSON-compatible session state projected through Gateway sessions                                                    |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | Durable exactly-once context injected into the next agent turn for one session                                                    |
| `api.registerTrustedToolPolicy(...)`                                                 | Bundled/trusted pre-plugin tool policy that can block or rewrite tool params                                                      |
| `api.registerToolMetadata(...)`                                                      | Tool catalog display metadata without changing the tool implementation                                                            |
| `api.registerCommand(...)`                                                           | Scoped plugin commands; command results can set `continueAgent: true`; Discord native commands support `descriptionLocalizations` |
| `api.session.controls.registerControlUiDescriptor(...)`                              | Control UI contribution descriptors for session, tool, run, or settings surfaces                                                  |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | Cleanup callbacks for plugin-owned runtime resources on reset/delete/reload paths                                                 |
| `api.agent.events.registerAgentEventSubscription(...)`                               | Sanitized event subscriptions for workflow state and monitors                                                                     |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | Per-run plugin scratch state cleared on terminal run lifecycle                                                                    |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | Cleanup metadata for plugin-owned scheduler jobs; does not schedule work or create task records                                   |
| `api.session.workflow.sendSessionAttachment(...)`                                    | Bundled-only host-mediated file attachment delivery to the active direct-outbound session route                                   |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | Bundled-only Cron-backed scheduled session turns plus tag-based cleanup                                                           |
| `api.session.controls.registerSessionAction(...)`                                    | Typed session actions clients can dispatch through the Gateway                                                                    |

Use the grouped namespaces for new plugin code:

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

The equivalent flat methods remain available as deprecated compatibility
aliases for existing plugins. Do not add new plugin code that calls
`api.registerSessionExtension`, `api.enqueueNextTurnInjection`,
`api.registerControlUiDescriptor`, `api.registerRuntimeLifecycle`,
`api.registerAgentEventSubscription`, `api.emitAgentEvent`,
`api.setRunContext`, `api.getRunContext`, `api.clearRunContext`,
`api.registerSessionSchedulerJob`, `api.registerSessionAction`,
`api.sendSessionAttachment`, `api.scheduleSessionTurn`, or
`api.unscheduleSessionTurnsByTag` directly.

`scheduleSessionTurn(...)` is a session-scoped convenience over the Gateway
Cron scheduler. Cron owns timing and creates the background task record when the
turn runs; the Plugin SDK only constrains the target session, plugin-owned
naming, and cleanup. Use `api.runtime.tasks.managedFlows` inside the scheduled
turn when the work itself needs durable multi-step Task Flow state.

The contracts intentionally split authority:

- External plugins can own session extensions, UI descriptors, commands, tool
  metadata, next-turn injections, and normal hooks.
- Trusted tool policies run before ordinary `before_tool_call` hooks and are
  bundled-only because they participate in host safety policy.
- Reserved command ownership is bundled-only. External plugins should use their
  own command names or aliases.
- `allowPromptInjection=false` disables prompt-mutating hooks including
  `agent_turn_prepare`, `before_prompt_build`, `heartbeat_prompt_contribution`,
  prompt fields from legacy `before_agent_start`, and
  `enqueueNextTurnInjection`.

Examples of non-Plan consumers:

| Plugin archetype             | Hooks used                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Approval workflow            | Session extension, command continuation, next-turn injection, UI descriptor                                                            |
| Budget/workspace policy gate | Trusted tool policy, tool metadata, session projection                                                                                 |
| Background lifecycle monitor | Runtime lifecycle cleanup, agent event subscription, session scheduler ownership/cleanup, heartbeat prompt contribution, UI descriptor |
| Setup or onboarding wizard   | Session extension, scoped commands, Control UI descriptor                  
---
domain: plugins
topic: "Plugin Architecture Overview"
type: concept
keywords:
  - plugin architecture
  - plugins system
  - plugin lifecycle
  - slots
  - hooks
  - plugin registry
source: plugins/architecture.md
---

This is the **deep architecture reference** for the OpenClaw plugin system. For practical guides, start with one of the focused pages below.

**Install and use plugins:** End-user guide for adding, enabling, and troubleshooting plugins.
  

**Building plugins:** First-plugin tutorial with the smallest working manifest.
  

**Channel plugins:** Build a messaging channel plugin.
  

**Provider plugins:** Build a model provider plugin.
  

**SDK overview:** Import map and registration API reference.
  

## Public capability model

Capabilities are the public **native plugin** model inside OpenClaw. Every native OpenClaw plugin registers against one or more capability types:

| Capability             | Registration method                              | Example plugins                      |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| Text inference         | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| CLI inference backend  | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| Speech                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| Realtime transcription | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Realtime voice         | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| Media understanding    | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| Image generation       | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Music generation       | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| Video generation       | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Web fetch              | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Web search             | `api.registerWebSearchProvider(...)`             | `google`                             |
| Channel / messaging    | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |
| Gateway discovery      | `api.registerGatewayDiscoveryService(...)`       | `bonjour`                            |

> **Note:** A plugin that registers zero capabilities but provides hooks, tools, discovery services, or background services is a **legacy hook-only** plugin. That pattern is still fully supported.


### External compatibility stance

The capability model is landed in core and used by bundled/native plugins today, but external plugin compatibility still needs a tighter bar than "it is exported, therefore it is frozen."

| Plugin situation                                  | Guidance                                                                                         |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Existing external plugins                         | Keep hook-based integrations working; this is the compatibility baseline.                        |
| New bundled/native plugins                        | Prefer explicit capability registration over vendor-specific reach-ins or new hook-only designs. |
| External plugins adopting capability registration | Allowed, but treat capability-specific helper surfaces as evolving unless docs mark them stable. |

Capability registration is the intended direction. Legacy hooks remain the safest no-breakage path for external plugins during the transition. Exported helper subpaths are not all equal — prefer narrow documented contracts over incidental helper exports.

### Plugin shapes

OpenClaw classifies every loaded plugin into a shape based on its actual registration behavior (not just static metadata):

### plain-capability

Registers exactly one capability type (for example a provider-only plugin like `mistral`).
  ### hybrid-capability

Registers multiple capability types (for example `openai` owns text inference, speech, media understanding, and image generation).
  ### hook-only

Registers only hooks (typed or custom), no capabilities, tools, commands, or services.
  ### non-capability

Registers tools, commands, services, or routes but no capabilities.
  Use `openclaw plugins inspect <id>` to see a plugin's shape and capability breakdown. See [CLI reference](/cli/plugins#inspect) for details.

### Legacy hooks

The `before_agent_start` hook remains supported as a compatibility path for hook-only plugins. Legacy real-world plugins still depend on it.

Direction:

- keep it working
- document it as legacy
- prefer `before_model_resolve` for model/provider override work
- prefer `before_prompt_build` for prompt mutation work
- remove only after real usage drops and fixture coverage proves migration safety

### Compatibility signals

When you run `openclaw doctor` or `openclaw plugins inspect <id>`, you may see one of these labels:

| Signal                     | Meaning                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | Config parses fine and plugins resolve                       |
| **compatibility advisory** | Plugin uses a supported-but-older pattern (e.g. `hook-only`) |
| **legacy warning**         | Plugin uses `before_agent_start`, which is deprecated        |
| **hard error**             | Config is invalid or plugin failed to load                   |

Neither `hook-only` nor `before_agent_start` will break your plugin today: `hook-only` is advisory, and `before_agent_start` only triggers a warning. These signals also appear in `openclaw status --all` and `openclaw plugins doctor`.

## Architecture overview

OpenClaw's plugin system has four layers:

**Manifest + discovery**

OpenClaw finds candidate plugins from configured paths, workspace roots, global plugin roots, and bundled plugins. Discovery reads native `openclaw.plugin.json` manifests plus supported bundle manifests first.
  
**Enablement + validation**

Core decides whether a discovered plugin is enabled, disabled, blocked, or selected for an exclusive slot such as memory.
  
**Runtime loading**

Native OpenClaw plugins are loaded in-process and register capabilities into a central registry. Packaged JavaScript loads through native `require`; third-party local source TypeScript is the emergency Jiti fallback. Compatible bundles are normalized into registry records without importing runtime code.
  
**Surface consumption**

The rest of OpenClaw reads the registry to expose tools, channels, provider setup, hooks, HTTP routes, CLI commands, and services.
  
For plugin CLI specifically, root command discovery is split in two phases:

- parse-time metadata comes from `registerCli(..., { descriptors: [...] })`
- the real plugin CLI module can stay lazy and register on first invocation

That keeps plugin-owned CLI code inside the plugin while still letting OpenClaw reserve root command names before parsing.

The important design boundary:

- manifest/config validation should work from **manifest/schema metadata** without executing plugin code
- native capability discovery may load trusted plugin entry code to build a non-activating registry snapshot
- native runtime behavior comes from the plugin module's `register(api)` path with `api.registrationMode === "full"`

That split lets OpenClaw validate config, explain missing/disabled plugins, and build UI/schema hints before the full runtime is active.

### Plugin metadata snapshot and lookup table

Gateway startup builds one `PluginMetadataSnapshot` for the current config snapshot. The snapshot is metadata-only: it stores the installed plugin index, manifest registry, manifest diagnostics, owner maps, a plugin id normalizer, and manifest records. It does not hold loaded plugin modules, provider SDKs, package contents, or runtime exports.

Plugin-aware config validation, startup auto-enable, and Gateway plugin bootstrap consume that snapshot instead of rebuilding manifest/index metadata independently. `PluginLookUpTable` is derived from the same snapshot and adds the startup plugin plan for the current runtime config.

After startup, Gateway keeps the current metadata snapshot as a replaceable runtime product. Repeated runtime provider discovery can borrow that snapshot instead of reconstructing the installed index and manifest registry for each provider-catalog pass. The snapshot is cleared or replaced on Gateway shutdown, config/plugin inventory changes, and installed index writes; callers fall back to the cold manifest/index path when no compatible current snapshot exists. Compatibility checks must include plugin discovery roots such as `plugins.load.paths` and the default agent workspace, because workspace plugins are part of the metadata scope.

The snapshot and lookup table keep repeated startup decisions on the fast path:

- channel ownership
- deferred channel startup
- startup plugin ids
- provider and CLI backend ownership
- setup provider, command alias, model catalog provider, and manifest contract ownership
- plugin config schema and channel config schema validation
- startup auto-enable decisions

The safety boundary is snapshot replacement, not mutation. Rebuild the snapshot when config, plugin inventory, install records, or persisted index policy changes. Do not treat it as a broad mutable global registry, and do not keep unbounded historical snapshots. Runtime plugin loading remains separate from metadata snapshots so stale runtime state cannot be hidden behind a metadata cache.

The cache rule is documented in [Plugin architecture internals](/plugins/architecture-internals#plugin-cache-boundary): manifest and discovery metadata are fresh unless a caller holds an explicit snapshot, lookup table, or manifest registry for the current flow. Hidden metadata caches and wall-clock TTLs are not part of plugin loading. Only runtime loader, module, and dependency-artifact caches may persist after code or installed artifacts are actually loaded.

Some cold-path callers still reconstruct manifest registries directly from the persisted installed plugin index instead of receiving a Gateway `PluginLookUpTable`. That path now reconstructs the registry on demand; prefer passing the current lookup table or an explicit manifest registry through runtime flows when a caller already has one.

### Activation planning

Activation planning is part of the control plane. Callers can ask which plugins are relevant to a concrete command, provider, channel, route, agent harness, or capability before loading broader runtime registries.

The planner keeps current manifest behavior compatible:

- `activation.*` fields are explicit planner hints
- `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools`, and hooks remain manifest ownership fallback
- the ids-only planner API stays available for existing callers
- the plan API reports reason labels so diagnostics can distinguish explicit hints from ownership fallback

> **Note:** Do not treat `activation` as a lifecycle hook or a replacement for `register(...)`. It is metadata used to narrow loading. Prefer ownership fields when they already describe the relationship; use `activation` only for extra planner hints.


### Channel plugins and the shared message tool

Channel plugins do not need to register a separate send/edit/react tool for normal chat actions. OpenClaw keeps one shared `message` tool in core, and channel plugins own the channel-specific discovery and execution behind it.

The current boundary is:

- core owns the shared `message` tool host, prompt wiring, session/thread bookkeeping, and execution dispatch
- channel plugins own scoped action discovery, capability discovery, and any channel-specific schema fragments
- channel plugins own provider-specific session conversation grammar, such as how conversation ids encode thread ids or inherit from parent conversations
- channel plugins execute the final action through their action adapter

For channel plugins, the SDK surface is `ChannelMessageActionAdapter.describeMessageTool(...)`. That unified discovery call lets a plugin return its visible actions, capabilities, and schema contributions together so those pieces do not drift apart.

When a channel-specific message-tool param carries a media source such as a local path or remote media URL, the plugin should also return `mediaSourceParams` from `describeMessageTool(...)`. Core uses that explicit list to apply sandbox path normalization and outbound media-access hints without hardcoding plugin-owned param names. Prefer action-scoped maps there, not one channel-wide flat list, so a profile-only media param does not get normalized on unrelated actions like `send`.

Core passes runtime scope into that discovery step. Important fields include:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- trusted inbound `requesterSenderId`

That matters for context-sensitive plugins. A channel can hide or expose message actions based on the active account, current room/thread/message, or trusted requester identity without hardcoding channel-specific branches in the core `message` tool.

This is why embedded-runner routing changes are still plugin work: the runner is responsible for forwarding the current chat/session identity into the plugin discovery boundary so the shared `message` tool exposes the right channel-owned surface for the current turn.

For channel-owned execution helpers, bundled plugins should keep the execution runtime inside their own extension modules. Core no longer owns the Discord, Slack, Telegram, or Wha
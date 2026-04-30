---
domain: plugins
topic: "Plugin Architecture: Types, Installation, and Configuration"
type: concept
keywords:
  - plugin architecture
  - plugin types
  - plugin installation
  - openclaw plugins
  - plugin config
related:
  - plugins/building-plugins
  - plugins/sdk-overview
  - plugins/plugin-manifest
source: plugins/architecture.md
---

OpenClaw plugins extend the Gateway with channels, providers, tools, and hooks. Plugins are TypeScript/JavaScript packages installed via npm or local paths.

## Plugin Types

| Type | Description |
|------|-------------|
| Channel plugin | Adds a messaging channel (Matrix, Teams, Mattermost, etc.) |
| Provider plugin | Adds an AI model provider |
| Tool plugin | Adds new agent tools |
| Hook plugin | Intercepts agent/gateway events |
| Memory plugin | Alternative memory backend |

## Plugin Installation

```bash
# Install from npm
openclaw plugins install @openclaw/matrix

# Install from local path
openclaw plugins install /path/to/my-plugin

# List installed plugins
openclaw plugins list
```

## Plugin Configuration

```json5
{
  plugins: {
    entries: {
      "plugin-id": {
        enabled: true,
        config: { /* plugin-specific config */ }
      }
    }
  }
}
```

This is the **deep architecture reference** for the OpenClaw plugin system. For practical guides, start with one of the focused pages below.

    End-user guide for adding, enabling, and troubleshooting plugins.

    First-plugin tutorial with the smallest working manifest.

    Build a messaging channel plugin.

    Build a model provider plugin.

    Import map and registration API reference.

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

A plugin that registers zero capabilities but provides hooks, tools, discovery services, or background services is a **legacy hook-only** plugin. That pattern is still fully supported.

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

    Registers exactly one capability type (for example a provider-only plugin like `mistral`).

    Registers multiple capability types (for example `openai` owns text inference, speech, media understanding, and image generation).

    Registers only hooks (typed or custom), no capabilities, tools, commands, or services.

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

| Signal

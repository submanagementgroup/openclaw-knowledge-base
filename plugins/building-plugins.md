---
domain: plugins
topic: "Building Plugins: Step-by-Step Guide to Creating OpenClaw Plugin Packages"
type: procedure
keywords:
  - building plugins
  - plugin development
  - create plugin
  - plugin package
  - plugin guide
related:
  - plugins/sdk-setup
  - plugins/sdk-overview
  - plugins/plugin-manifest
source:
  - plugins/building-plugins.md
  - plugins/building-extensions.md
---

Step-by-step guide to building a plugin for OpenClaw.

Plugins extend OpenClaw with new capabilities: channels, model providers,
speech, realtime transcription, realtime voice, media understanding, image
generation, video generation, web fetch, web search, agent tools, or any
combination.

You do not need to add your plugin to the OpenClaw repository. Publish to
[ClawHub](/tools/clawhub) and users install with
`openclaw plugins install <package-name>`. OpenClaw tries ClawHub first and
falls back to npm automatically for packages that still use npm distribution.

## Prerequisites

- Node >= 22 and a package manager (npm or pnpm)
- Familiarity with TypeScript (ESM)
- For in-repo plugins: repository cloned and `pnpm install` done

## What kind of plugin?

    Connect OpenClaw to a messaging platform (Discord, IRC, etc.)

    Add a model provider (LLM, proxy, or custom endpoint)

    Register agent tools, event hooks, or services — continue below

For a channel plugin that isn't guaranteed to be installed when onboarding/setup
runs, use `createOptionalChannelSetupSurface(...)` from
`openclaw/plugin-sdk/channel-setup`. It produces a setup adapter + wizard pair
that advertises the install requirement and fails closed on real config writes
until the plugin is installed.

## Quick start: tool plugin

This walkthrough creates a minimal plugin that registers an agent tool. Channel
and provider plugins have dedicated guides linked above.

    ```json package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "description": "Adds a custom tool to OpenClaw",
      "activation": {
        "onStartup": true
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    Every plugin needs a manifest, even with no config, and every plugin should
    declare `activation.onStartup` intentionally. Runtime-registered tools need
    startup import, so this example sets it to `true`. See
    [Manifest](/plugins/manifest) for the full schema. The canonical ClawHub
    publish snippets live in `docs/snippets/plugin-publish/`.

    ```typescript
    // index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { Type } from "@sinclair/typebox";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Do a thing",
          parameters: Type.Object({ input: Type.String() }),
          async execute(_id, params) {
            return { content: [{ type: "text", text: `Got: ${params.input}` }] };
          },
        });
      },
    });
    ```

    `definePluginEntry` is for non-channel plugins. For channels, use
    `defineChannelPluginEntry` — see [Channel Plugins](/plugins/sdk-channel-plugins).
    For full entry point options, see [Entry Points](/plugins/sdk-entrypoints).

    **External plugins:** validate and publish with ClawHub, then install:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    OpenClaw also checks ClawHub before npm for bare package specs like
    `@myorg/openclaw-my-plugin`; npm remains a fallback for packages that have
    not migrated to ClawHub yet.

    **In-repo plugins:** place under the bundled plugin workspace tree — automatically discovered.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

## Plugin capabilities

A single plugin can register any number of capabilities via the `api` object:

| Capability             | Registration method                              | Detailed guide                                                                  |
| ---------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| Text inference (LLM)   | `api.registerProvider(...)`                      | [Provider Plugins](/plugins/sdk-provider-plugins)                               |
| CLI inference backend  | `api.registerCliBackend(...)`                    | [CLI Backends](/gateway/cli-backends)                                           |
| Channel / messaging    | `api.registerChannel(...)`                       | [Channel Plugins](/plugins/sdk-channel-plugins)                                 |
| Speech (TTS/STT)       | `api.registerSpeechProvider(...)`                | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Realtime transcription | `api.registerRealtimeTranscriptionProvider(...)` | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Realtime voice         | `api.registerRealtimeVoiceProvider(...)`         | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Media understanding    | `api.registerMediaUnderstandingProvider(...)`    | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Image generation       | `api.registerImageGenerationProvider(...)`       | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Music generation       | `api.registerMusicGenerationProvider(...)`       | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Video generation       | `api.registerVideoGenerationProvider(...)`       | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web fetch              | `api.registerWebFetchProvider(...)`              | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web search             | `api.registerWebSearchProvider(...)`             | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Tool-result middleware | `api.registerAgentToolResultMiddleware(...)`     | [SDK Overview](/plugins/sdk-overview#registration-api)                          |
| Agent tools            | `api.registerTool(...)`                          | Below                                                                           |
| Custom commands        | `api.registerCommand(...)`                       | [Entry Points](/plugins/sdk-entrypoints)                                        |
| Plugin hooks           | `api.on(...)`                                    | [Plugin hooks](/plugins/hooks)                                       

## Building Extensions

This page has moved to [Building Plugins](/plugins/building-plugins).

## Related

- [Building plugins](/plugins/building-plugins)
- [Plugin architecture](/plugins/architecture)

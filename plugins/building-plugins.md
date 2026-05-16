---
domain: plugins
topic: "Building Plugins: Developer Guide"
type: procedure
keywords:
  - building plugins
  - create plugin
  - plugin development
  - custom plugin
  - plugin authoring
source: plugins/building-plugins.md
---

Plugins extend OpenClaw with new capabilities: channels, model providers,
speech, realtime transcription, realtime voice, media understanding, image
generation, video generation, web fetch, web search, agent tools, or any
combination.

You do not need to add your plugin to the OpenClaw repository. Publish to
[ClawHub](/clawhub) and users install with
`openclaw plugins install clawhub:<package-name>`. Bare package specs still
install from npm during the launch cutover.

## Prerequisites

- Node >= 22 and a package manager (npm or pnpm)
- Familiarity with TypeScript (ESM)
- For in-repo plugins: repository cloned and `pnpm install` done. Source
  checkout plugin development is pnpm-only because OpenClaw loads bundled
  plugins from the `extensions/*` workspace packages.

## What kind of plugin?

**Channel plugin:** Connect OpenClaw to a messaging platform (Discord, IRC, etc.)
  

**Provider plugin:** Add a model provider (LLM, proxy, or custom endpoint)
  

**CLI backend plugin:** Map a local AI CLI into OpenClaw's text fallback runner
  

**Tool / hook plugin:** Register agent tools, event hooks, or services - continue below
  

For a channel plugin that isn't guaranteed to be installed when onboarding/setup
runs, use `createOptionalChannelSetupSurface(...)` from
`openclaw/plugin-sdk/channel-setup`. It produces a setup adapter + wizard pair
that advertises the install requirement and fails closed on real config writes
until the plugin is installed.

## Quick start: tool plugin

This walkthrough creates a minimal plugin that registers an agent tool. Channel
and provider plugins have dedicated guides linked above.

**Create the package and manifest**


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
      "contracts": {
        "tools": ["my_tool"]
      },
      "activation": {
        "onStartup": true
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    

    Every plugin needs a manifest, even with no config. Runtime-registered tools
    must be listed in `contracts.tools` so OpenClaw can discover the owning
    plugin without loading every plugin runtime. Plugins should also declare
    `activation.onStartup` intentionally. This example sets it to `true`. See
    [Manifest](/plugins/manifest) for the full schema. The canonical ClawHub
    publish snippets live in `docs/snippets/plugin-publish/`.

  
**Write the entry point**

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
    `defineChannelPluginEntry` - see [Channel Plugins](/plugins/sdk-channel-plugins).
    For full entry point options, see [Entry Points](/plugins/sdk-entrypoints).

  
**Test and publish**

**External plugins:** validate and publish with ClawHub, then install:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    Bare package specs like `@myorg/openclaw-my-plugin` install from npm during
    the launch cutover. Use `clawhub:` when you want ClawHub resolution.

    **In-repo plugins:** place under the bundled plugin workspace tree - automatically discovered.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

  
## Plugin capabilities

A single plugin can register any number of capabilities via the `api` object:

| Capability             | Registration method                              | Detailed guide                                                                  |
| ---------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| Text inference (LLM)   | `api.registerProvider(...)`                      | [Provider Plugins](/plugins/sdk-provider-plugins)                               |
| CLI inference backend  | `api.registerCliBackend(...)`                    | [CLI Backend Plugins](/plugins/cli-backend-plugins)                             |
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
| Plugin hooks           | `api.on(...)`                                    | [Plugin hooks](/plugins/hooks)                                                  |
| Internal event hooks   | `api.registerHook(...)`                          | [Entry Points](/plugins/sdk-entrypoints)                                        |
| HTTP routes            | `api.registerHttpRoute(...)`                     | [Internals](/plugins/architecture-internals#gateway-http-routes)                |
| CLI subcommands        | `api.registerCli(...)`                           | [Entry Points](/plugins/sdk-entrypoints)                                        |

For the full registration API, see [SDK Overview](/plugins/sdk-overview#registration-api).

Bundled plugins can use `api.registerAgentToolResultMiddleware(...)` when they
need async tool-result rewriting before the model sees the output. Declare the
targeted runtimes in `contracts.agentToolResultMiddleware`, for example
`["pi", "codex"]`. This is a trusted bundled-plugin seam; external
plugins should prefer regular OpenClaw plugin hooks unless OpenClaw grows an
explicit trust policy for this capability.

If your plugin registers custom gateway RPC methods, keep them on a
plugin-specific prefix. Core admin namespaces (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) stay reserved and always resolve to
`operator.admin`, even if a plugin asks for a narrower scope.

Hook guard semantics to keep in mind:

- `before_tool_call`: `{ block: true }` is terminal and stops lower-priority handlers.
- `before_tool_call`: `{ block: false }` is treated as no decision.
- `before_tool_call`: `{ requireApproval: true }` pauses agent execution and prompts the user for approval via the exec approval overlay, Telegram buttons, Discord interactions, or the `/approve` command on any channel.
- `before_install`: `{ block: true }` is terminal and stops lower-priority handlers.
- `before_install`: `{ block: false }` is treated as no decision.
- `message_sending`: `{ cancel: true }` is terminal and stops lower-priority handlers.
- `message_sending`: `{ cancel: false }` is treated as no decision.
- `message_received`: prefer the typed `threadId` field when you need inbound thread/topic routing. Keep `metadata` for channel-specific extras.
- `message_sending`: prefer typed `replyToId` / `threadId` routing fields over channel-specific metadata keys.

The `/approve` command handles both exec and plugin approvals with bounded fallback: when an exec approval id is not found, OpenClaw retries the same id through plugin approvals. Plugin approval forwarding can be configured independently via `approvals.plugin` in config.

If custom approval plumbing needs to detect that same bounded fallback case,
prefer `isApprovalNotFoundError` from `openclaw/plugin-sdk/error-runtime`
instead of matching approval-expiry strings manually.

See [Plugin hooks](/plugins/hooks) for examples and the hook reference.

## Registering agent tools

Tools are typed functions the LLM can call. They can be required (always
available) or optional (user opt-in):

```typescript
register(api) {
  // Required tool - always available
  api.registerTool({
    name: "my_tool",
    description: "Do a thing",
    parameters: Type.Object({ input: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  // Optional tool - user must add to allowlist
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      async execute(_id, params) {
        return { content: [{ type: "text", text: params.pipeline }] };
      },
    },
    { optional: true },
  );
}
```

Tool factories receive a runtime-supplied context object. Use
`ctx.activeModel` when a tool needs to log, display, or adapt to the active
model for the current turn. The object can include `provider`, `modelId`, and
`modelRef`. Treat it as informational runtime metadata, not as a security
boundary against the local operator, installed plugin code, or a modified
OpenClaw runtime. For sensitive local tools, keep an explicit plugin or operator
opt-in and fail closed when the active model metadata is missing or unsuitable.

Every tool registered with `api.registerTool(...)` must also be declared in the
plugin manifest:

```json
{
  "contracts": {
    "tools": ["my_tool", "workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

OpenClaw captures and caches the validated descriptor from the registered tool,
so plugins do not duplicate `description` or schema data in the manifest.
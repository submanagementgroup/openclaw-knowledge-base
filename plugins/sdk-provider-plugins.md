---
domain: plugins
topic: "Provider Plugin Development"
type: reference
keywords:
  - provider plugin
  - provider SDK
  - model provider plugin
  - custom provider
  - SDK provider
source: plugins/sdk-provider-plugins.md
---

This guide walks through building a provider plugin that adds a model provider
(LLM) to OpenClaw. By the end you will have a provider with a model catalog,
API key auth, and dynamic model resolution.

> **Note:** If you have not built any OpenClaw plugin before, read
  [Getting Started](/plugins/building-plugins) first for the basic package
  structure and manifest setup.


> **Note:** Provider plugins add models to OpenClaw's normal inference loop. If the model
  must run through a native agent daemon that owns threads, compaction, or tool
  events, pair the provider with an [agent harness](/plugins/sdk-agent-harness)
  instead of putting daemon protocol details in core.


## Walkthrough

**Package and manifest**

### Step 1: Package and manifest

    
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
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
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI model provider",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "providerAuthEnvVars": {
        "acme-ai": ["ACME_AI_API_KEY"]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API key",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API key"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    

    The manifest declares `providerAuthEnvVars` so OpenClaw can detect
    credentials without loading your plugin runtime. Add `providerAuthAliases`
    when a provider variant should reuse another provider id's auth. `modelSupport`
    is optional and lets OpenClaw auto-load your provider plugin from shorthand
    model ids like `acme-large` before runtime hooks exist. If you publish the
    provider on ClawHub, those `openclaw.compat` and `openclaw.build` fields
    are required in `package.json`.

  
**Register the provider**

A minimal text provider needs an `id`, `label`, `auth`, and `catalog`.
    `catalog` is the provider-owned runtime/config hook; it can call live
    vendor APIs and returns `models.providers` entries.

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` is the newer control-plane catalog surface
    for list/help/picker UI. Use it for text, image-generation,
    video-generation, and music-generation rows. Keep vendor endpoint calls and
    response mapping in the plugin; OpenClaw owns the shared row shape, source
    labels, and help rendering.

    That is a working provider. Users can now
    `openclaw onboard --acme-ai-api-key <key>` and select
    `acme-ai/acme-large` as their model.

    If the upstream provider uses different control tokens than OpenClaw, add a
    small bidirectional text transform instead of replacing the stream path:

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` rewrites the final system prompt and text message content before
    transport. `output` rewrites assistant text deltas and final text before
    OpenClaw parses its own control markers or channel delivery.

    For bundled providers that only register one text provider with API-key
    auth plus a single catalog-backed runtime, prefer the narrower
    `defineSingleProviderPluginEntry(...)` helper:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API key",
            hint: "API key from your Acme AI dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Enter your Acme AI API key",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` is the live catalog path used when OpenClaw can resolve real
    provider auth. It may perform provider-specific discovery. Use
    `buildStaticProvider` only for offline rows that are safe to show before auth
    is configured; it must not require credentials or make network requests.
    OpenClaw's `models list --all` display currently executes static catalogs
    only for bundled provider plugins, with an empty config, empty env, and no
    agent/workspace paths.

    If your auth flow also needs to patch `models.providers.*`, aliases, and
    the agent default model during onboarding, use the preset helpers from
    `openclaw/plugin-sdk/provider-onboard`. The narrowest helpers are
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)`, and
    `createModelCatalogPresetAppliers(...)`.

    When a provider's native endpoint supports streamed usage blocks on the
    normal `openai-completions` transport, prefer the shared catalog helpers in
    `openclaw/plugin-sdk/provider-catalog-shared` instead of hardcoding
    provider-id checks. `supportsNativeStreamingUsageCompat(...)` and
    `applyProviderNativeStreamingUsageCompat(...)` detect support from the
    endpoint capability map, so native Moonshot/DashScope-style endpoints still
    opt in even when a plugin is using a custom provider id.

  
**Add dynamic model resolution**

If your provider accepts arbitrary model IDs (like a proxy or router),
    add `resolveDynamicModel`:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    If resolving requires a network call, use `prepareDynamicModel` for async
    warm-up - `resolveDynamicModel` runs again after it completes.

  
**Add runtime hooks (as needed)**

Most providers only need `catalog` + `resolveDynamicModel`. Add hooks
    incrementally as your provider requires them.

    Shared helper builders now cover the most common replay/tool-compat
    families, so plugins usually do not need to hand-wire each hook one by one:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    Available replay families today:

    | Family | What it wires in | Bundled examples |
    | --- | --- | --- |
    | `openai-compatible` | Shared OpenAI-style replay policy for OpenAI-compatible transports, including tool-call-id sanitation, assistant-first ordering fixes, and generic Gemini-turn validation where the transport needs it | `moonshot`, `ollama`, `xai`, `zai` |
    | `anthropic-by-model` | Claude-aware replay policy chosen by `modelId`, so Anthropic-message transports only get Claude-specific thinking-block cleanup when the resolved model is actually a Claude id | `amazon-bedrock`, `anthropic-vertex` |
    | `google-gemini` | Native Gemini replay policy plus bootstrap replay sanitation and tagged reasoning-output mode | `google`, `google-gemini-cli` |
    | `passthrough-gemini` | Gemini thought-signature sanitation for Gemini models running through OpenAI-compatible proxy transports; does not enable native Gemini replay validation or bootstrap rewrites | `openrouter`, `kilocode`, `opencode`, `opencode-go` |
    | `hybrid-anthropic-openai` | Hybrid policy for providers that mix Anthropic-message and OpenAI-compatible model surfaces in one plugin; optional Claude-only thinking-block dropping stays scoped to the Anthropic side | `minimax` |

    Available stream families today:

    | Family | What it wires in | Bundled examples |
    | --- | --- | --- |
    | `google-thinking` | Gemini thinking payload normalization on the shared stream path | `google`, `google-gemini-cli` |
    | `kilocode-thinking` | Kilo reasoning wrapper on the shared proxy stream path, with `kilo/auto` and unsupported proxy reasoning ids skipping injected thinking | `kilocode` |
    | `moonshot-thinking` | Moonshot binary native-thinking payload mapping from config + `/think` level | `moonshot` |
    | `minimax-fast-mode` | MiniMax fast-mode model rewrite on the shared stream path | `minimax`, `minimax-portal` |
    | `openai-responses-defaults` | Shared native OpenAI/Codex Responses wrappers: attribution headers, `/fast`/`serviceTier`, text verbosity, native Codex web search, reasoning-compat payload shaping, and Responses context management | `openai`, `openai-codex` |
    | `openrouter-thinking` | OpenRouter reasoning wrapper for proxy routes, with unsupported-model/`auto` skips handled centrally | `openrouter` |
    | `tool-stream-default-on` | Default-on `tool_stream` wrapper for providers like Z.AI that w
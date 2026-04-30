---
domain: plugins
topic: "SDK Provider Plugins: Building Custom AI Model Provider Integrations"
type: procedure
keywords:
  - provider plugins
  - SDK provider
  - model provider plugin
  - custom provider
  - provider implementation
related:
  - plugins/sdk-overview
  - plugins/plugin-manifest
source: plugins/sdk-provider-plugins.md
---

Building provider plugins with the OpenClaw SDK. Provider plugins add new AI model backends.

This guide walks through building a provider plugin that adds a model provider
(LLM) to OpenClaw. By the end you will have a provider with a model catalog,
API key auth, and dynamic model resolution.

  If you have not built any OpenClaw plugin before, read
  [Getting Started](/plugins/building-plugins) first for the basic package
  structure and manifest setup.

  Provider plugins add models to OpenClaw's normal inference loop. If the model
  must run through a native agent daemon that owns threads, compaction, or tool
  events, pair the provider with an [agent harness](/plugins/sdk-agent-harness)
  instead of putting daemon protocol details in core.

## Walkthrough

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

    A minimal provider needs an `id`, `label`, `auth`, and `catalog`:

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
      },
    });
    ```

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

    If your provider accepts arbitrary model IDs (like a proxy or router),
    add `resolveDynamicModel`:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId

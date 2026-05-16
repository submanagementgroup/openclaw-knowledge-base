---
domain: plugins
topic: "Plugin Manifest Reference"
type: reference
keywords:
  - plugin manifest
  - manifest.json
  - plugin.json
  - manifest schema
  - plugin definition
  - manifest fields
source: plugins/manifest.md
---

This page is for the **native OpenClaw plugin manifest** only.

For compatible bundle layouts, see [Plugin bundles](/plugins/bundles).

Compatible bundle formats use different manifest files:

- Codex bundle: `.codex-plugin/plugin.json`
- Claude bundle: `.claude-plugin/plugin.json` or the default Claude component
  layout without a manifest
- Cursor bundle: `.cursor-plugin/plugin.json`

OpenClaw auto-detects those bundle layouts too, but they are not validated
against the `openclaw.plugin.json` schema described here.

For compatible bundles, OpenClaw currently reads bundle metadata plus declared
skill roots, Claude command roots, Claude bundle `settings.json` defaults,
Claude bundle LSP defaults, and supported hook packs when the layout matches
OpenClaw runtime expectations.

Every native OpenClaw plugin **must** ship a `openclaw.plugin.json` file in the
**plugin root**. OpenClaw uses this manifest to validate configuration
**without executing plugin code**. Missing or invalid manifests are treated as
plugin errors and block config validation.

See the full plugin system guide: [Plugins](/tools/plugin).
For the native capability model and current external-compatibility guidance:
[Capability model](/plugins/architecture#public-capability-model).

## What this file does

`openclaw.plugin.json` is the metadata OpenClaw reads **before it loads your
plugin code**. Everything below must be cheap enough to inspect without booting
plugin runtime.

**Use it for:**

- plugin identity, config validation, and config UI hints
- auth, onboarding, and setup metadata (alias, auto-enable, provider env vars, auth choices)
- activation hints for control-plane surfaces
- shorthand model-family ownership
- static capability-ownership snapshots (`contracts`)
- QA runner metadata the shared `openclaw qa` host can inspect
- channel-specific config metadata merged into catalog and validation surfaces

**Do not use it for:** registering runtime behavior, declaring code entrypoints,
or npm install metadata. Those belong in your plugin code and `package.json`.

## Minimal example

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Rich example

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter provider plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API key",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API key",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Top-level field reference

| Field                                | Required | Type                             | What it means                                                                                                                                                                                                                       |
| ------------------------------------ | -------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                 | Yes      | `string`                         | Canonical plugin id. This is the id used in `plugins.entries.<id>`.                                                                                                                                                                 |
| `configSchema`                       | Yes      | `object`                         | Inline JSON Schema for this plugin's config.                                                                                                                                                                                        |
| `enabledByDefault`                   | No       | `true`                           | Marks a bundled plugin as enabled by default. Omit it, or set any non-`true` value, to leave the plugin disabled by default.                                                                                                        |
| `enabledByDefaultOnPlatforms`        | No       | `string[]`                       | Marks a bundled plugin as enabled by default only on the listed Node.js platforms, for example `["darwin"]`. Explicit config still wins.                                                                                            |
| `legacyPluginIds`                    | No       | `string[]`                       | Legacy ids that normalize to this canonical plugin id.                                                                                                                                                                              |
| `autoEnableWhenConfiguredProviders`  | No       | `string[]`                       | Provider ids that should auto-enable this plugin when auth, config, or model refs mention them.                                                                                                                                     |
| `kind`                               | No       | `"memory"` \| `"context-engine"` | Declares an exclusive plugin kind used by `plugins.slots.*`.                                                                                                                                                                        |
| `channels`                           | No       | `string[]`                       | Channel ids owned by this plugin. Used for discovery and config validation.                                                                                                                                                         |
| `providers`                          | No       | `string[]`                       | Provider ids owned by this plugin.                                                                                                                                                                                                  |
| `providerCatalogEntry`               | No       | `string`                         | Lightweight provider-catalog module path, relative to the plugin root, for manifest-scoped provider catalog metadata that can be loaded without activating the full plugin runtime.                                                 |
| `modelSupport`                       | No       | `object`                         | Manifest-owned shorthand model-family metadata used to auto-load the plugin before runtime.                                                                                                                                         |
| `modelCatalog`                       | No       | `object`                         | Declarative model catalog metadata for providers owned by this plugin. This is the control-plane contract for future read-only listing, onboarding, model pickers, aliases, and suppression without loading plugin runtime.         |
| `modelPricing`                       | No       | `object`                         | Provider-owned external pricing lookup policy. Use it to opt local/self-hosted providers out of remote pricing catalogs or map provider refs to OpenRouter/LiteLLM catalog ids without hardcoding provider ids in core.             |
| `modelIdNormalization`               | No       | `object`                         | Provider-owned model-id alias/prefix cleanup that must run before provider runtime loads.                                                                                                                                           |
| `providerEndpoints`                  | No       | `object[]`                       | Manifest-owned endpoint host/baseUrl metadata for provider routes that core must classify before provider runtime loads.                                                                                                            |
| `providerRequest`                    | No       | `object`                         | Cheap provider-family and request-compatibility metadata used by generic request policy before provider runtime loads.                                                                                                              |
| `cliBackends`                        | No       | `string[]`                       | CLI inference backend ids owned by this plugin. Used for startup auto-activation from explicit config refs.                                                                                                                         |
| `syntheticAuthRefs`                  | No       | `string[]`                       | Provider or CLI backend refs whose plugin-owned synthetic auth hook should be probed during cold model discovery before runtime loads.                                                                                              |
| `nonSecretAuthMarkers`               | No       | `string[]`                       | Bundled-plugin-owned placeholder API key values that represent non-secret local, OAuth, or ambient credential state.                                                                                                                |
| `commandAliases`                     | No       | `object[]`                       | Command names owned by this plugin that should produce plugin-aware config and CLI diagnostics before runtime loads.                                                                                                                |
| `providerAuthEnvVars`                | No       | `Record<string, string[]>`       | Deprecated compatibility env metadata for provider auth/status lookup. Prefer `setup.providers[].envVars` for new plugins; OpenClaw still reads this during the deprecation window.                                                 |
| `providerAuthAliases`                | No       | `Record<string, string>`         | Provider ids that should reuse another provider id for auth lookup, for example a coding provider that shares the base provider API key and auth profiles.                                                                          |
| `channelEnvVars`                     | No       | `Record<string, string[]>`       | Cheap channel env metadata that OpenClaw can inspect without loading plugin code. Use this for env-driven channel setup or auth surfaces that generic startup/config helpers should see.                                            |
| `providerAuthChoices`                | No       | `object[]`                       | Cheap auth-choice metadata for onboarding pickers, preferred-provider resolution, and simple CLI flag wiring.                                                                                                                       |
| `activation`                         | No       | `object`                         | Cheap activation planner metadata for startup, provider, command, channel, route, and capability-triggered loading. Metadata only; plugin runtime still owns actual behavior.                                                       |
| `setup`                              | No       | `object`                         | Cheap setup/onboarding descriptors that discovery and setup surfaces can inspect without loading plugin runtime.                                                                                                                    |
| `qaRunners`                          | No       | `object[]`                       | Cheap QA runner descriptors used by the shared `openclaw qa` host before plugin runtime loads.                                                                                                                                      |
| `contracts`                          | No       | `object`                         | Static capability ownership snapshot for external auth hooks, speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-generation, web-fetch, web search, and tool ownership. |
| `mediaUnderstandingProviderMetadata` | No       | `Record<string, object>`         | Cheap media-understanding defaults for provider ids declared in `contracts.mediaUnderstandingProviders`.                                                                                                                            |
| `imageGenerationProviderMetadata`    | No       | `Record<string, object>`         | Cheap image-generation auth metadata for provider ids declared in `contracts.imageGenerationProviders`, including provider-owned auth aliases and base-url guards.  
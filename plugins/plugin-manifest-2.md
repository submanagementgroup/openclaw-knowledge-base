---
domain: plugins
topic: "Plugin Manifest: Model Support, Model Catalog, Provider Endpoints, and Pricing"
type: reference
keywords:
  - modelSupport
  - modelCatalog
  - providerEndpoints
  - modelPricing
  - manifest validation
related:
  - plugins/plugin-manifest
source: plugins/manifest.md
---

Plugin manifest continued: model support, model catalog, provider endpoints, pricing, and validation.

                                                                                                                             |
| `kind`                               | No       | `"memory"` \| `"context-engine"` | Declares an exclusive plugin kind used by `plugins.slots.*`.                                                                                                                                                                      |
| `channels`                           | No       | `string[]`                       | Channel ids owned by this plugin. Used for discovery and config validation.                                                                                                                                                       |
| `providers`                          | No       | `string[]`                       | Provider ids owned by this plugin.                                                                                                                                                                                                |
| `providerDiscoveryEntry`             | No       | `string`                         | Lightweight provider-discovery module path, relative to the plugin root, for manifest-scoped provider catalog metadata that can be loaded without activating the full plugin runtime.                                             |
| `modelSupport`                       | No       | `object`                         | Manifest-owned shorthand model-family metadata used to auto-load the plugin before runtime.                                                                                                                                       |
| `modelCatalog`                       | No       | `object`                         | Declarative model catalog metadata for providers owned by this plugin. This is the control-plane contract for future read-only listing, onboarding, model pickers, aliases, and suppression without loading plugin runtime.       |
| `modelPricing`                       | No       | `object`                         | Provider-owned external pricing lookup policy. Use it to opt local/self-hosted providers out of remote pricing catalogs or map provider refs to OpenRouter/LiteLLM catalog ids without hardcoding provider ids in core.           |
| `modelIdNormalization`               | No       | `object`                         | Provider-owned model-id alias/prefix cleanup that must run before provider runtime loads.                                                                                                                                         |
| `providerEndpoints`                  | No       | `object[]`                       | Manifest-owned endpoint host/baseUrl metadata for provider routes that core must classify before provider runtime loads.                                                                                                          |
| `providerRequest`                    | No       | `object`                         | Cheap provider-family and request-compatibility metadata used by generic request policy before provider runtime loads.                                                                                                            |
| `cliBackends`                        | No       | `string[]`                       | CLI inference backend ids owned by this plugin. Used for startup auto-activation from explicit config refs.                                                                                                                       |
| `syntheticAuthRefs`                  | No       | `string[]`                       | Provider or CLI backend refs whose plugin-owned synthetic auth hook should be probed during cold model discovery before runtime loads.                                                                                            |
| `nonSecretAuthMarkers`               | No       | `string[]`                       | Bundled-plugin-owned placeholder API key values that represent non-secret local, OAuth, or ambient credential state.                                                                                                              |
| `commandAliases`                     | No       | `object[]`                       | Command names owned by this plugin that should produce plugin-aware config and CLI diagnostics before runtime loads.                                                                                                              |
| `providerAuthEnvVars`                | No       | `Record<string, string[]>`       | Deprecated compatibility env metadata for provider auth/status lookup. Prefer `setup.providers[].envVars` for new plugins; OpenClaw still reads this during the deprecation window.                                               |
| `providerAuthAliases`                | No       | `Record<string, string>`         | Provider ids that should reuse another provider id for auth lookup, for example a coding provider that shares the base provider API key and auth profiles.                                                                        |
| `channelEnvVars`                     | No       | `Record<string, string[]>`       | Cheap channel env metadata that OpenClaw can inspect without loading plugin code. Use this for env-driven channel setup or auth surfaces that generic startup/config helpers should see.                                          |
| `providerAuthChoices`                | No       | `object[]`                       | Cheap auth-choice metadata for onboarding pickers, preferred-provider resolution, and simple CLI flag wiring.                                                                                                                     |
| `activation`                         | No       | `object`                         | Cheap activation planner metadata for startup, provider, command, channel, route, and capability-triggered loading. Metadata only; plugin runtime still owns actual behavior.                                                     |
| `setup`                              | No       | `object`                         | Cheap setup/onboarding descriptors that discovery and setup surfaces can inspect without loading plugin runtime.                                                                                                                  |
| `qaRunners`                          | No       | `object[]`                       | Cheap QA runner descriptors used by the shared `openclaw qa` host before plugin runtime loads.                                                                                                                                    |
| `contracts`                          | No       | `object`                         | Static bundled capability snapshot for external auth hooks, speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-

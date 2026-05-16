---
domain: plugins
topic: "Plugin Manifest Extended Reference"
type: reference
keywords:
  - manifest
  - plugin manifest advanced
  - manifest hooks
  - manifest slots
  - manifest contracts
source: plugins/manifest.md
---

                                                                |
| `videoGenerationProviderMetadata`    | No       | `Record<string, object>`         | Cheap video-generation auth metadata for provider ids declared in `contracts.videoGenerationProviders`, including provider-owned auth aliases and base-url guards.                                                                  |
| `musicGenerationProviderMetadata`    | No       | `Record<string, object>`         | Cheap music-generation auth metadata for provider ids declared in `contracts.musicGenerationProviders`, including provider-owned auth aliases and base-url guards.                                                                  |
| `toolMetadata`                       | No       | `Record<string, object>`         | Cheap availability metadata for plugin-owned tools declared in `contracts.tools`. Use it when a tool should not load runtime unless config, env, or auth evidence exists.                                                           |
| `channelConfigs`                     | No       | `Record<string, object>`         | Manifest-owned channel config metadata merged into discovery and validation surfaces before runtime loads.                                                                                                                          |
| `skills`                             | No       | `string[]`                       | Skill directories to load, relative to the plugin root.                                                                                                                                                                             |
| `name`                               | No       | `string`                         | Human-readable plugin name.                                                                                                                                                                                                         |
| `description`                        | No       | `string`                         | Short summary shown in plugin surfaces.                                                                                                                                                                                             |
| `version`                            | No       | `string`                         | Informational plugin version.                                                                                                                                                                                                       |
| `uiHints`                            | No       | `Record<string, object>`         | UI labels, placeholders, and sensitivity hints for config fields.                                                                                                                                                                   |

## Generation provider metadata reference

The generation provider metadata fields describe static auth signals for
providers declared in the matching `contracts.*GenerationProviders` list.
OpenClaw reads these fields before provider runtime loads so core tools can
decide whether a generation provider is available without importing every
provider plugin.

Use these fields only for cheap, declarative facts. Transport, request
transforms, token refresh, credential validation, and actual generation behavior
stay in the plugin runtime.

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

Each metadata entry supports:

| Field           | Required | Type       | What it means                                                                                                                       |
| --------------- | -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`       | No       | `string[]` | Additional provider ids that should count as static auth aliases for the generation provider.                                       |
| `authProviders` | No       | `string[]` | Provider ids whose configured auth profiles should count as auth for this generation provider.                                      |
| `configSignals` | No       | `object[]` | Cheap config-only availability signals for local or self-hosted providers that can be configured without auth profiles or env vars. |
| `authSignals`   | No       | `object[]` | Explicit auth signals. When present, these replace the default signal set from the provider id, `aliases`, and `authProviders`.     |

Each `configSignals` entry supports:

| Field         | Required | Type       | What it means                                                                                                                                                                           |
| ------------- | -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`    | Yes      | `string`   | Dot path to the plugin-owned config object to inspect, for example `plugins.entries.example.config`.                                                                                    |
| `overlayPath` | No       | `string`   | Dot path inside the root config whose object should overlay the root object before evaluating the signal. Use this for capability-specific config such as `image`, `video`, or `music`. |
| `required`    | No       | `string[]` | Dot paths inside the effective config that must have configured values. Strings must be non-empty; objects and arrays must not be empty.                                                |
| `requiredAny` | No       | `string[]` | Dot paths inside the effective config where at least one must have a configured value.                                                                                                  |
| `mode`        | No       | `object`   | Optional string mode guard inside the effective config. Use this when config-only availability applies only to one mode.                                                                |

Each `mode` guard supports:

| Field        | Required | Type       | What it means                                                                      |
| ------------ | -------- | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | No       | `string`   | Dot path inside the effective config. Defaults to `mode`.                          |
| `default`    | No       | `string`   | Mode value to use when the config omits the path.                                  |
| `allowed`    | No       | `string[]` | If present, the signal passes only when the effective mode is one of these values. |
| `disallowed` | No       | `string[]` | If present, the signal fails when the effective mode is one of these values.       |

Each `authSignals` entry supports:

| Field             | Required | Type     | What it means                                                                                                                                                                 |
| ----------------- | -------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Yes      | `string` | Provider id to check in configured auth profiles.                                                                                                                             |
| `providerBaseUrl` | No       | `object` | Optional guard that makes the signal count only when the referenced configured provider uses an allowed base URL. Use this when an auth alias is valid only for certain APIs. |

Each `providerBaseUrl` guard supports:

| Field             | Required | Type       | What it means                                                                                                                                        |
| ----------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Yes      | `string`   | Provider config id whose `baseUrl` should be checked.                                                                                                |
| `defaultBaseUrl`  | No       | `string`   | Base URL to assume when the provider config omits `baseUrl`.                                                                                         |
| `allowedBaseUrls` | Yes      | `string[]` | Allowed base URLs for this auth signal. The signal is ignored when the configured or default base URL does not match one of these normalized values. |

## Tool metadata reference

`toolMetadata` uses the same `configSignals` and `authSignals` shapes as
generation provider metadata, keyed by tool name. `contracts.tools` declares
ownership. `toolMetadata` declares cheap availability evidence so OpenClaw can
avoid importing a plugin runtime just to have its tool factory return `null`.

```json
{
  "providerAuthEnvVars": {
    "example": ["EXAMPLE_API_KEY"]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

If a tool has no `toolMetadata`, OpenClaw preserves the existing behavior and
loads the owning plugin when the tool contract matches policy. For hot-path
tools whose factory depends on auth/config, plugin authors should declare
`toolMetadata` instead of making core import runtime to ask.

## providerAuthChoices reference

Each `providerAuthChoices` entry describes one onboarding or auth choice.
OpenClaw reads this before provider runtime loads.
Provider setup lists use these manifest choices, descriptor-derived setup
choices, and install-catalog metadata without loading provider runtime.

| Field                 | Required | Type                                            | What it means                                                                                            |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider`            | Yes      | `string`                                        | Provider id this choice belongs to.                                                                      |
| `method`              | Yes      | `string`                                        | Auth method id to dispatch to.                                                                           |
| `choiceId`            | Yes      | `string`                                        | Stable auth-choice id used by onboarding and CLI flows.                                                  |
| `choiceLabel`         | No       | `string`                                        | User-facing label. If omitted, OpenClaw falls back to `choiceId`.                                        |
| `choiceHint`          | No       | `string`                                        | Short helper text for the picker.                                                                        |
| `assistantPriority`   | No       | `number`                                        | Lower values sort earlier in assistant-driven interactive pickers.                                       |
| `assistantVisibility` | No       | `"visible"` \| `"manual-only"`                  | Hide the choice from assistant pickers while still allowing manual CLI selection.                        |
| `deprecatedChoiceIds` | No       | `string[]`                                      | Legacy choice ids that should redirect users to this replacement choice.                                 |
| `groupId`             | No       | `string`                                        | Optional group id for grouping related choices.                                                          |
| `groupLabel`          | No       | `string`                                        | User-facing label for that group.                                                                        |
| `groupHint`           | No       | `string`                                        | Short helper text for the group.                                                                         |
| `optionKey`           | No       | `string`                                        | Internal option key for simple one-flag auth flows.                                                      |

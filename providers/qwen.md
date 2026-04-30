---
domain: providers
topic: "Alibaba Qwen Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Alibaba Qwen
  - qwen provider
  - AI provider
  - model config
  - API key
  - Qwen
  - Alibaba
  - Tongyi
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/qwen.md
---

Alibaba Qwen provider setup, configuration, and model reference for OpenClaw.

**Qwen OAuth has been removed.** The free-tier OAuth integration
(`qwen-portal`) that used `portal.qwen.ai` endpoints is no longer available.
See [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) for
background.

OpenClaw now treats Qwen as a first-class bundled provider with canonical id
`qwen`. The bundled provider targets the Qwen Cloud / Alibaba DashScope and
Coding Plan endpoints and keeps legacy `modelstudio` ids working as a
compatibility alias.

- Provider: `qwen`
- Preferred env var: `QWEN_API_KEY`
- Also accepted for compatibility: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- API style: OpenAI-compatible

If you want `qwen3.6-plus`, prefer the **Standard (pay-as-you-go)** endpoint.
Coding Plan support can lag behind the public catalog.

## Getting started

Choose your plan type and follow the setup steps.

    **Best for:** subscription-based access through the Qwen Coding Plan.

        Create or copy an API key from [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys).

        For the **Global** endpoint:

        ```bash
        openclaw onboard --auth-choice qwen-api-key
        ```

        For the **China** endpoint:

        ```bash
        openclaw onboard --auth-choice qwen-api-key-cn
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider qwen
        ```

    Legacy `modelstudio-*` auth-choice ids and `modelstudio/...` model refs still
    work as compatibility aliases, but new setup flows should prefer the canonical
    `qwen-*` auth-choice ids and `qwen/...` model refs. If you define an exact
    custom `models.providers.modelstudio` entry with another `api` value, that
    custom provider owns `modelstudio/...` refs instead of the Qwen compatibility
    alias.

    **Best for:** pay-as-you-go access through the Standard Model Studio endpoint, including models like `qwen3.6-plus` that may not be available on the Coding Plan.

        Create or copy an API key from [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys).

        For the **Global** endpoint:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key
        ```

        For the **China** endpoint:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key-cn
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider qwen
        ```

    Legacy `modelstudio-*` auth-choice ids and `modelstudio/...` model refs still
    work as compatibility aliases, but new setup flows should prefer the canonical
    `qwen-*` auth-choice ids and `qwen/...` model refs. If you define an exact
    custom `models.providers.modelstudio` entry with another `api` value, that
    custom provider owns `modelstudio/...` refs instead of the Qwen compatibility
    alias.

## Plan types and endpoints

| Plan                       | Region | Auth choice                | Endpoint                                         |
| -------------------------- | ------ | -------------------------- | ------------------------------------------------ |
| Standard (pay-as-you-go)   | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pay-as-you-go)   | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (subscription) | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (subscription) | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

The provider auto-selects the endpoint based on your auth choice. Canonical
choices use the `qwen-*` family; `modelstudio-*` remains compatibility-only.
You can override with a custom `baseUrl` in config.

**Manage keys:** [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) |
**Docs:** [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## Built-in catalog

OpenClaw currently ships this bundled Qwen catalog. The configured catalog is
endpoint-aware: Coding Plan configs omit models that are only known to work on
the Standard endpoint.

| Model ref                   | Input       | Context   | Notes                                              |
| --------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`         | text, image | 1,000,000 | Default model                                      |
| `qwen/qwen3.6-plus`         | text, image | 1,000,000 | Prefer Standard endpoints when you need this model |
| `qwen/qwen3-max-2026-01-23` | text        | 262,144   | Qwen Max line                                      |
| `qwen/qwen3-coder-next`     | text        | 262,144   | Coding                                             |
| `qwen/qwen3-coder-plus`     | text        | 1,000,000 | Coding                                             |
| `qwen/MiniMax-M2.5`         | text        | 1,000,000 | Reasoning enabled                                  |
| `qwen/glm-5`                | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`              | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`            | text, image | 262,144   | Moonshot AI via Alibaba                            |

Availability can still vary by endpoint and billing plan even when a model is
present in the bundled catalog.

## Thinking Controls

For reasoning-enabled Qwen Cloud models, the bundled provider maps OpenClaw
thinking levels to DashScope's top-level `enable_thinking` request flag. Disabled
thinking sends `enable_thinking: false`; other thinking levels send
`enable_thinking: true`.

## Multimodal add-ons

The `qwen` plugin also exposes multimodal capabilities on the **Standard**
DashScope endpoints (not the Coding Plan endpoints):

- **Video understanding** via `qwen-vl-max-latest`
- **Wan video generation** via `wan2.6-t2v` (default), `wan2.6-i2v`, `wan2.6-r2v`, `wan2.6-r2v-flash`, `wan2.7-r2v`

To use Qwen as the default video provider:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

See [Video Generation](/tools/video-generation) for shared tool parameters, provider selection, and failover behavior.

## Advanced configuration

    The bundled Qwen plugin registers media understanding for images and video
    on the **Standard** DashScope endpoints (not the Coding Plan endpoints).

    | Property      | Value                 |
    | ------------- | --------------------- |
    | Model         | `qwen-vl-max-latest`  |
    | Supported input | Images, video       |

    Media understanding is auto-resolved from the configured Qwen auth — no
    additional config is needed. Ensure you are using a Standard (pay-as-you-go)
    endpoint for media understanding support.

    `qwen3.6-plus` is available on the Standard (pay-as-you-go) Model Studio
    endpoints:

    - China: `dashscope.aliyuncs.com/compatible-mode/v1`
    - Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

    If the Coding Plan endpoints return an "unsupported model" error for
    `qwen3.6-plus`, switch to Standard (pay-as-you-go) instead of the Coding Plan
    endpoint/key pair.

    OpenClaw's bundled Qwen catalog does not advertise `qwen3.6-plus` on Coding
    Plan endpoints, but explicitly configured `qwen/qwen3.6-plus` entries under
    `models.providers.qwen.models` are honored on Coding Plan baseUrls so you
    can opt that model in if Aliyun enables it on your subscription. The
    upstream API still decid

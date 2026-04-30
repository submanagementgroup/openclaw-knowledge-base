---
domain: providers
topic: "Alibaba Cloud Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Alibaba Cloud
  - alibaba provider
  - AI provider
  - model config
  - API key
  - Alibaba Cloud
  - Alibaba provider
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/alibaba.md
---

Alibaba Cloud provider setup, configuration, and model reference for OpenClaw.

OpenClaw ships a bundled `alibaba` video-generation provider for Wan models on
Alibaba Model Studio / DashScope.

- Provider: `alibaba`
- Preferred auth: `MODELSTUDIO_API_KEY`
- Also accepted: `DASHSCOPE_API_KEY`, `QWEN_API_KEY`
- API: DashScope / Model Studio async video generation

## Getting started

    ```bash
    openclaw onboard --auth-choice qwen-standard-api-key
    ```

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "alibaba/wan2.6-t2v",
          },
        },
      },
    }
    ```

    ```bash
    openclaw models list --provider alibaba
    ```

Any of the accepted auth keys (`MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`, `QWEN_API_KEY`) will work. The `qwen-standard-api-key` onboarding choice configures the shared DashScope credential.

## Built-in Wan models

The bundled `alibaba` provider currently registers:

| Model ref                  | Mode                      |
| -------------------------- | ------------------------- |
| `alibaba/wan2.6-t2v`       | Text-to-video             |
| `alibaba/wan2.6-i2v`       | Image-to-video            |
| `alibaba/wan2.6-r2v`       | Reference-to-video        |
| `alibaba/wan2.6-r2v-flash` | Reference-to-video (fast) |
| `alibaba/wan2.7-r2v`       | Reference-to-video        |

## Current limits

| Parameter             | Limit                                                     |
| --------------------- | --------------------------------------------------------- |
| Output videos         | Up to **1** per request                                   |
| Input images          | Up to **1**                                               |
| Input videos          | Up to **4**                                               |
| Duration              | Up to **10 seconds**                                      |
| Supported controls    | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Reference image/video | Remote `http(s)` URLs only                                |

Reference image/video mode currently requires **remote http(s) URLs**. Local file paths are not supported for reference inputs.

## Advanced configuration

    The bundled `qwen` provider also uses Alibaba-hosted DashScope endpoints for
    Wan video generation. Use:

    - `qwen/...` when you want the canonical Qwen provider surface
    - `alibaba/...` when you want the direct vendor-owned Wan video surface

    See the [Qwen provider docs](/providers/qwen) for more detail.

    OpenClaw checks for auth keys in this order:

    1. `MODELSTUDIO_API_KEY` (preferred)
    2. `DASHSCOPE_API_KEY`
    3. `QWEN_API_KEY`

    Any of these will authenticate the `alibaba` provider.

## Related

    Shared video tool parameters and provider selection.

    Qwen provider setup and DashScope integration.

    Agent defaults and model configuration.

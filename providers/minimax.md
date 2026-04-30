---
domain: providers
topic: "MiniMax Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - MiniMax
  - minimax provider
  - AI provider
  - model config
  - API key
  - MiniMax
  - Chinese AI
  - MiniMax provider
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/minimax.md
---

MiniMax provider setup, configuration, and model reference for OpenClaw.

OpenClaw's MiniMax provider defaults to **MiniMax M2.7**.

MiniMax also provides:

- Bundled speech synthesis via T2A v2
- Bundled image understanding via `MiniMax-VL-01`
- Bundled music generation via `music-2.6`
- Bundled `web_search` through the MiniMax Coding Plan search API

Provider split:

| Provider ID      | Auth    | Capabilities                                                                                        |
| ---------------- | ------- | --------------------------------------------------------------------------------------------------- |
| `minimax`        | API key | Text, image generation, music generation, video generation, image understanding, speech, web search |
| `minimax-portal` | OAuth   | Text, image generation, music generation, video generation, image understanding, speech             |

## Built-in catalog

| Model                    | Type             | Description                              |
| ------------------------ | ---------------- | ---------------------------------------- |
| `MiniMax-M2.7`           | Chat (reasoning) | Default hosted reasoning model           |
| `MiniMax-M2.7-highspeed` | Chat (reasoning) | Faster M2.7 reasoning tier               |
| `MiniMax-VL-01`          | Vision           | Image understanding model                |
| `image-01`               | Image generation | Text-to-image and image-to-image editing |
| `music-2.6`              | Music generation | Default music model                      |
| `music-2.5`              | Music generation | Previous music generation tier           |
| `music-2.0`              | Music generation | Legacy music generation tier             |
| `MiniMax-Hailuo-2.3`     | Video generation | Text-to-video and image reference flows  |

## Getting started

Choose your preferred auth method and follow the setup steps.

    **Best for:** quick setup with MiniMax Coding Plan via OAuth, no API key required.

            ```bash
            openclaw onboard --auth-choice minimax-global-oauth
            ```

            This authenticates against `api.minimax.io`.

            ```bash
            openclaw models list --provider minimax-portal
            ```

            ```bash
            openclaw onboard --auth-choice minimax-cn-oauth
            ```

            This authenticates against `api.minimaxi.com`.

            ```bash
            openclaw models list --provider minimax-portal
            ```

    OAuth setups use the `minimax-portal` provider id. Model refs follow the form `minimax-portal/MiniMax-M2.7`.

    Referral link for MiniMax Coding Plan (10% off): [MiniMax Coding Plan](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)

    **Best for:** hosted MiniMax with Anthropic-compatible API.

            ```bash
            openclaw onboard --auth-choice minimax-global-api
            ```

            This configures `api.minimax.io` as the base URL.

            ```bash
            openclaw models list --provider minimax
            ```

            ```bash
            openclaw onboard --auth-choice minimax-cn-api
            ```

            This configures `api.minimaxi.com` as the base URL.

            ```bash
            openclaw models list --provider minimax
            ```

    ### Config example

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "minimax/MiniMax-M2.7" } } },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M2.7",
                name: "MiniMax M2.7",
                reasoning: true,
                input: ["text"],
                cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
                contextWindow: 204800,
                maxTokens: 131072,
              },
              {
                id: "MiniMax-M2.7-highspeed",
                name: "MiniMax M2.7 Highspeed",
                reasoning: true,
                input: ["text"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
                contextWindow: 204800,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    On the Anthropic-compatible streaming path, OpenClaw disables MiniMax thinking by default unless you explicitly set `thinking` yourself. MiniMax's streaming endpoint emits `reasoning_content` in OpenAI-style delta chunks instead of native Anthropic thinking blocks, which can leak internal reasoning into visible output if left enabled implicitly.

    API-key setups use the `minimax` provider id. Model refs follow the form `minimax/MiniMax-M2.7`.

## Configure via `openclaw configure`

Use the interactive config wizard to set MiniMax without editing JSON:

    ```bash
    openclaw configure
    ```

    Choose **Model/auth** from the menu.

    Pick one of the available MiniMax options:

    | Auth choice | Description |
    | --- | --- |
    | `minimax-global-oauth` | International OAuth (Coding Plan) |
    | `minimax-cn-oauth` | China OAuth (Coding Plan) |
    | `minimax-global-api` | International API key |
    | `minimax-cn-api` | China API key |

    Select your default model when prompted.

## Capabilities

### Image generation

The MiniMax plugin registers the `image-01` model for the `image_generate` tool. It supports:

- **Text-to-image generation** with aspect ratio control
- **Image-to-image editing** (subject reference) with aspect ratio control
- Up to **9 output images** per request
- Up to **1 reference image** per edit request
- Supported aspect ratios: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`

To use MiniMax for image generation, set it as the image generation provider:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

The plugin uses the same `MINIMAX_API_KEY` or OAuth auth as the text models. No additional configuration is needed if MiniMax is already set up.

Both `minimax` and `minimax-portal` register `image_generate` with the same
`image-01` model. API-key setups use `MINIMAX_API_KEY`; OAuth setups can use
the bundled `minimax-portal` auth path instead.

Image generation always uses MiniMax's dedicated image endpoint
(`/v1/image_generation`) and ignores `models.providers.minimax.baseUrl`,
since that field configures the chat/Anthropic-compatible base URL. Set
`MINIMAX_API_HOST=https://api.minimaxi.com` to route image generation
through the CN endpoint; the default global endpoint is
`https://api.minimax.io`.

When onboarding or API-key setup writes explicit `models.providers.minimax`
entries, OpenClaw materializes `MiniMax-M2.7` and
`MiniMax-M2.7-highspeed` as text-only chat models. Image understanding is
exposed separately through the plugin-owned `MiniMax-VL-01` media provider.

See [Image Generation](/tools/image-generation) for shared tool parameters, provider selection, and failover behavior.

### Text-to-speech

The bundled `minimax` plugin registers MiniMax T2A v2 as a speech provider for
`messages.tts`.

- Default TTS model: `speech-2.8-hd`
- Default voice: `English_expressive_narrator`
- Supported bundled model ids include `speech-2.8-hd`, `speech-2.8-turbo`,
  `speech-2.6-hd`, `speech-2.6-turbo`, `speech-02-hd`,
  `speech-02-turbo`, `speech-01-hd`, and `speech-01-turbo`.
- Auth resolution is `messages.tts.providers.minimax.apiKey`, then
  `minimax-portal` OAuth/token auth profiles, then Token Plan environment
  keys (`MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`,
  `MINIMAX_CODING_API_KEY`), then `MINIMAX_API_KEY`.
- If no TTS host is configured, OpenClaw reuses the configured
  `minimax-portal` OAuth host and strips Anthropic-compatible path suffixes
  such as `/anthropic`.
- Normal

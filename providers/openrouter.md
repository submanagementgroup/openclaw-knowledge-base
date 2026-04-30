---
domain: providers
topic: "OpenRouter Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - OpenRouter
  - openrouter provider
  - AI provider
  - model config
  - API key
  - OpenRouter
  - model aggregator
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/openrouter.md
---

OpenRouter provider setup, configuration, and model reference for OpenClaw.

OpenRouter provides a **unified API** that routes requests to many models behind a single
endpoint and API key. It is OpenAI-compatible, so most OpenAI SDKs work by switching the base URL.

## Getting started

    Create an API key at [openrouter.ai/keys](https://openrouter.ai/keys).

    ```bash
    openclaw onboard --auth-choice openrouter-api-key
    ```

    Onboarding defaults to `openrouter/auto`. Pick a concrete model later:

    ```bash
    openclaw models set openrouter/<provider>/<model>
    ```

## Config example

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## Model references

Model refs follow the pattern `openrouter/<provider>/<model>`. For the full list of
available providers and models, see [/concepts/model-providers](/concepts/model-providers).

Bundled fallback examples:

| Model ref                         | Notes                        |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | OpenRouter automatic routing |
| `openrouter/moonshotai/kimi-k2.6` | Kimi K2.6 via MoonshotAI     |

## Image generation

OpenRouter can also back the `image_generate` tool. Use an OpenRouter image model under `agents.defaults.imageGenerationModel`:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw sends image requests to OpenRouter's chat completions image API with `modalities: ["image", "text"]`. Gemini image models receive supported `aspectRatio` and `resolution` hints through OpenRouter's `image_config`. Use `agents.defaults.imageGenerationModel.timeoutMs` for slower OpenRouter image models; the `image_generate` tool's per-call `timeoutMs` parameter still wins.

## Video generation

OpenRouter can also back the `video_generate` tool through its asynchronous `/videos` API. Use an OpenRouter video model under `agents.defaults.videoGenerationModel`:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw submits text-to-video and image-to-video jobs to OpenRouter, polls
the returned `polling_url`, and downloads the completed video from
OpenRouter's `unsigned_urls` or the documented job content endpoint.
Reference images are sent as first/last frame images by default; images
tagged with `reference_image` are sent as OpenRouter input references. The
bundled `google/veo-3.1-fast` default advertises the currently supported 4/6/8
second durations, `720P`/`1080P` resolutions, and `16:9`/`9:16` aspect
ratios. Video-to-video is not registered for OpenRouter because the upstream
video generation API currently accepts text and image references.

## Text-to-speech

OpenRouter can also be used as a TTS provider through its OpenAI-compatible
`/audio/speech` endpoint.

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openrouter",
      providers: {
        openrouter: {
          model: "hexgrad/kokoro-82m",
          voice: "af_alloy",
          responseFormat: "mp3",
        },
      },
    },
  },
}
```

If `messages.tts.providers.openrouter.apiKey` is omitted, TTS reuses
`models.providers.openrouter.apiKey`, then `OPENROUTER_API_KEY`.

## Authentication and headers

OpenRouter uses a Bearer token with your API key under the hood.

On real OpenRouter requests (`https://openrouter.ai/api/v1`), OpenClaw also adds
OpenRouter's documented app-attribution headers:

| Header                    | Value                 |
| ------------------------- | --------------------- |
| `HTTP-Referer`            | `https://openclaw.ai` |
| `X-OpenRouter-Title`      | `OpenClaw`            |
| `X-OpenRouter-Categories` | `cli-agent`           |

If you repoint the OpenRouter provider at some other proxy or base URL, OpenClaw
does **not** inject those OpenRouter-specific headers or Anthropic cache markers.

## Advanced configuration

    On verified OpenRouter routes, Anthropic model refs keep the
    OpenRouter-specific Anthropic `cache_control` markers that OpenClaw uses for
    better prompt-cache reuse on system/developer prompt blocks.

    On supported non-`auto` routes, OpenClaw maps the selected thinking level to
    OpenRouter proxy reasoning payloads. Unsupported model hints and
    `openrouter/auto` skip that reasoning injection. Hunter Alpha also skips
    proxy reasoning for stale configured model refs because OpenRouter could
    return final answer text in reasoning fields for that retired route.

    OpenRouter still runs through the proxy-style OpenAI-compatible path, so
    native OpenAI-only request shaping such as `serviceTier`, Responses `store`,
    OpenAI reasoning-compat payloads, and prompt-cache hints is not forwarded.

    Gemini-backed OpenRouter refs stay on the proxy-Gemini path: OpenClaw keeps
    Gemini thought-signature sanitation there, but does not enable native Gemini
    replay validation or bootstrap rewrites.

    If you pass OpenRouter provider routing under model params, OpenClaw forwards
    it as OpenRouter routing metadata before the shared stream wrappers run.

## Related

    Choosing providers, model refs, and failover behavior.

    Full config reference for agents, models, and providers.

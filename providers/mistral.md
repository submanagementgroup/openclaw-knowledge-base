---
domain: providers
topic: "Mistral Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Mistral
  - mistral provider
  - AI provider
  - model config
  - API key
  - Mistral
  - mistral-large
  - European AI
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/mistral.md
---

Mistral provider setup, configuration, and model reference for OpenClaw.

OpenClaw supports Mistral for both text/image model routing (`mistral/...`) and
audio transcription via Voxtral in media understanding.
Mistral can also be used for memory embeddings (`memorySearch.provider = "mistral"`).

- Provider: `mistral`
- Auth: `MISTRAL_API_KEY`
- API: Mistral Chat Completions (`https://api.mistral.ai/v1`)

## Getting started

    Create an API key in the [Mistral Console](https://console.mistral.ai/).

    ```bash
    openclaw onboard --auth-choice mistral-api-key
    ```

    Or pass the key directly:

    ```bash
    openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
    ```

    ```json5
    {
      env: { MISTRAL_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
    }
    ```

    ```bash
    openclaw models list --provider mistral
    ```

## Built-in LLM catalog

OpenClaw currently ships this bundled Mistral catalog:

| Model ref                        | Input       | Context | Max output | Notes                                                            |
| -------------------------------- | ----------- | ------- | ---------- | ---------------------------------------------------------------- |
| `mistral/mistral-large-latest`   | text, image | 262,144 | 16,384     | Default model                                                    |
| `mistral/mistral-medium-2508`    | text, image | 262,144 | 8,192      | Mistral Medium 3.1                                               |
| `mistral/mistral-small-latest`   | text, image | 128,000 | 16,384     | Mistral Small 4; adjustable reasoning via API `reasoning_effort` |
| `mistral/pixtral-large-latest`   | text, image | 128,000 | 32,768     | Pixtral                                                          |
| `mistral/codestral-latest`       | text        | 256,000 | 4,096      | Coding                                                           |
| `mistral/devstral-medium-latest` | text        | 262,144 | 32,768     | Devstral 2                                                       |
| `mistral/magistral-small`        | text        | 128,000 | 40,000     | Reasoning-enabled                                                |

## Audio transcription (Voxtral)

Use Voxtral for batch audio transcription through the media understanding
pipeline.

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

The media transcription path uses `/v1/audio/transcriptions`. The default audio model for Mistral is `voxtral-mini-latest`.

## Voice Call streaming STT

The bundled `mistral` plugin registers Voxtral Realtime as a Voice Call
streaming STT provider.

| Setting      | Config path                                                            | Default                                 |
| ------------ | ---------------------------------------------------------------------- | --------------------------------------- |
| API key      | `plugins.entries.voice-call.config.streaming.providers.mistral.apiKey` | Falls back to `MISTRAL_API_KEY`         |
| Model        | `...mistral.model`                                                     | `voxtral-mini-transcribe-realtime-2602` |
| Encoding     | `...mistral.encoding`                                                  | `pcm_mulaw`                             |
| Sample rate  | `...mistral.sampleRate`                                                | `8000`                                  |
| Target delay | `...mistral.targetStreamingDelayMs`                                    | `800`                                   |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "mistral",
            providers: {
              mistral: {
                apiKey: "${MISTRAL_API_KEY}",
                targetStreamingDelayMs: 800,
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw defaults Mistral realtime STT to `pcm_mulaw` at 8 kHz so Voice Call
can forward Twilio media frames directly. Use `encoding: "pcm_s16le"` and a
matching `sampleRate` only if your upstream stream is already raw PCM.

## Advanced configuration

    `mistral/mistral-small-latest` maps to Mistral Small 4 and supports [adjustable reasoning](https://docs.mistral.ai/capabilities/reasoning/adjustable) on the Chat Completions API via `reasoning_effort` (`none` minimizes extra thinking in the output; `high` surfaces full thinking traces before the final answer).

    OpenClaw maps the session **thinking** level to Mistral's API:

    | OpenClaw thinking level                          | Mistral `reasoning_effort` |
    | ------------------------------------------------ | -------------------------- |
    | **off** / **minimal**                            | `none`                     |
    | **low** / **medium** / **high** / **xhigh** / **adaptive** / **max** | `high`     |

    Other bundled Mistral catalog models do not use this parameter. Keep using `magistral-*` models when you want Mistral's native reasoning-first behavior.

    Mistral can serve memory embeddings via `/v1/embeddings` (default model: `mistral-embed`).

    ```json5
    {
      memorySearch: { provider: "mistral" },
    }
    ```

    - Mistral auth uses `MISTRAL_API_KEY`.
    - Provider base URL defaults to `https://api.mistral.ai/v1`.
    - Onboarding default model is `mistral/mistral-large-latest`.
    - Z.AI uses Bearer auth with your API key.

## Related

    Choosing providers, model refs, and failover behavior.

    Audio transcription setup and provider selection.

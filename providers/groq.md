---
domain: providers
topic: "Groq Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Groq
  - groq provider
  - AI provider
  - model config
  - API key
  - groq
  - fast inference
  - llama groq
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/groq.md
---

Groq provider setup, configuration, and model reference for OpenClaw.

[Groq](https://groq.com) provides ultra-fast inference on open-source models
(Llama, Gemma, Mistral, and more) using custom LPU hardware. OpenClaw connects
to Groq through its OpenAI-compatible API.

| Property | Value             |
| -------- | ----------------- |
| Provider | `groq`            |
| Auth     | `GROQ_API_KEY`    |
| API      | OpenAI-compatible |

## Getting started

    Create an API key at [console.groq.com/keys](https://console.groq.com/keys).

    ```bash
    export GROQ_API_KEY="gsk_..."
    ```

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "groq/llama-3.3-70b-versatile" },
        },
      },
    }
    ```

### Config file example

```json5
{
  env: { GROQ_API_KEY: "gsk_..." },
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## Built-in catalog

Groq's model catalog changes frequently. Run `openclaw models list | grep groq`
to see currently available models, or check
[console.groq.com/docs/models](https://console.groq.com/docs/models).

| Model                       | Notes                              |
| --------------------------- | ---------------------------------- |
| **Llama 3.3 70B Versatile** | General-purpose, large context     |
| **Llama 3.1 8B Instant**    | Fast, lightweight                  |
| **Gemma 2 9B**              | Compact, efficient                 |
| **Mixtral 8x7B**            | MoE architecture, strong reasoning |

Use `openclaw models list --provider groq` for the most up-to-date list of
models available on your account.

## Reasoning models

OpenClaw maps its shared `/think` levels to Groq's model-specific
`reasoning_effort` values. For `qwen/qwen3-32b`, disabled thinking sends
`none` and enabled thinking sends `default`. For Groq GPT-OSS reasoning models,
OpenClaw sends `low`, `medium`, or `high`; disabled thinking omits
`reasoning_effort` because those models do not support a disabled value.

## Audio transcription

Groq also provides fast Whisper-based audio transcription. When configured as a
media-understanding provider, OpenClaw uses Groq's `whisper-large-v3-turbo`
model to transcribe voice messages through the shared `tools.media.audio`
surface.

```json5
{
  tools: {
    media: {
      audio: {
        models: [{ provider: "groq" }],
      },
    },
  },
}
```

    | Property | Value |
    |----------|-------|
    | Shared config path | `tools.media.audio` |
    | Default base URL   | `https://api.groq.com/openai/v1` |
    | Default model      | `whisper-large-v3-turbo` |
    | API endpoint       | OpenAI-compatible `/audio/transcriptions` |

    If the Gateway runs as a daemon (launchd/systemd), make sure `GROQ_API_KEY` is
    available to that process (for example, in `~/.openclaw/.env` or via
    `env.shellEnv`).

    Keys set only in your interactive shell are not visible to daemon-managed
    gateway processes. Use `~/.openclaw/.env` or `env.shellEnv` config for
    persistent availability.

## Related

    Choosing providers, model refs, and failover behavior.

    Full config schema including provider and audio settings.

    Groq dashboard, API docs, and pricing.

    Official Groq model catalog.

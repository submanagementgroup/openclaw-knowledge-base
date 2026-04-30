---
domain: providers
topic: "Cohere Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Cohere
  - arcee provider
  - AI provider
  - model config
  - API key
  - Arcee
  - custom models
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/arcee.md
---

Cohere provider setup, configuration, and model reference for OpenClaw.

[Arcee AI](https://arcee.ai) provides access to the Trinity family of mixture-of-experts models through an OpenAI-compatible API. All Trinity models are Apache 2.0 licensed.

Arcee AI models can be accessed directly via the Arcee platform or through [OpenRouter](/providers/openrouter).

| Property | Value                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| Provider | `arcee`                                                                               |
| Auth     | `ARCEEAI_API_KEY` (direct) or `OPENROUTER_API_KEY` (via OpenRouter)                   |
| API      | OpenAI-compatible                                                                     |
| Base URL | `https://api.arcee.ai/api/v1` (direct) or `https://openrouter.ai/api/v1` (OpenRouter) |

## Getting started

        Create an API key at [Arcee AI](https://chat.arcee.ai/).

        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        Create an API key at [OpenRouter](https://openrouter.ai/keys).

        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        The same model refs work for both direct and OpenRouter setups (for example `arcee/trinity-large-thinking`).

## Non-interactive setup

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```

## Built-in catalog

OpenClaw currently ships this bundled Arcee catalog:

| Model ref                      | Name                   | Input | Context | Cost (in/out per 1M) | Notes                                     |
| ------------------------------ | ---------------------- | ----- | ------- | -------------------- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | text  | 256K    | $0.25 / $0.90        | Default model; reasoning enabled          |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | text  | 128K    | $0.25 / $1.00        | General-purpose; 400B params, 13B active  |
| `arcee/trinity-mini`           | Trinity Mini 26B       | text  | 128K    | $0.045 / $0.15       | Fast and cost-efficient; function calling |

The onboarding preset sets `arcee/trinity-large-thinking` as the default model.

## Supported features

| Feature                                       | Supported                    |
| --------------------------------------------- | ---------------------------- |
| Streaming                                     | Yes                          |
| Tool use / function calling                   | Yes                          |
| Structured output (JSON mode and JSON schema) | Yes                          |
| Extended thinking                             | Yes (Trinity Large Thinking) |

    If the Gateway runs as a daemon (launchd/systemd), make sure `ARCEEAI_API_KEY`
    (or `OPENROUTER_API_KEY`) is available to that process (for example, in
    `~/.openclaw/.env` or via `env.shellEnv`).

    When using Arcee models via OpenRouter, the same `arcee/*` model refs apply.
    OpenClaw handles routing transparently based on your auth choice. See the
    [OpenRouter provider docs](/providers/openrouter) for OpenRouter-specific
    configuration details.

## Related

    Access Arcee models and many others through a single API key.

    Choosing providers, model refs, and failover behavior.

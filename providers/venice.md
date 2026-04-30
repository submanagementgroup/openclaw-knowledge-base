---
domain: providers
topic: "Venice.ai Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Venice.ai
  - venice provider
  - AI provider
  - model config
  - API key
  - Venice
  - privacy-first
  - uncensored AI
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/venice.md
---

Venice.ai provider setup, configuration, and model reference for OpenClaw.

Venice AI provides **privacy-focused AI inference** with support for uncensored models and access to major proprietary models through their anonymized proxy. All inference is private by default — no training on your data, no logging.

## Why Venice in OpenClaw

- **Private inference** for open-source models (no logging).
- **Uncensored models** when you need them.
- **Anonymized access** to proprietary models (Opus/GPT/Gemini) when quality matters.
- OpenAI-compatible `/v1` endpoints.

## Privacy modes

Venice offers two privacy levels — understanding this is key to choosing your model:

| Mode           | Description                                                                                                                       | Models                                                        |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Private**    | Fully private. Prompts/responses are **never stored or logged**. Ephemeral.                                                       | Llama, Qwen, DeepSeek, Kimi, MiniMax, Venice Uncensored, etc. |
| **Anonymized** | Proxied through Venice with metadata stripped. The underlying provider (OpenAI, Anthropic, Google, xAI) sees anonymized requests. | Claude, GPT, Gemini, Grok                                     |

Anonymized models are **not** fully private. Venice strips metadata before forwarding, but the underlying provider (OpenAI, Anthropic, Google, xAI) still processes the request. Choose **Private** models when full privacy is required.

## Features

- **Privacy-focused**: Choose between "private" (fully private) and "anonymized" (proxied) modes
- **Uncensored models**: Access to models without content restrictions
- **Major model access**: Use Claude, GPT, Gemini, and Grok via Venice's anonymized proxy
- **OpenAI-compatible API**: Standard `/v1` endpoints for easy integration
- **Streaming**: Supported on all models
- **Function calling**: Supported on select models (check model capabilities)
- **Vision**: Supported on models with vision capability
- **No hard rate limits**: Fair-use throttling may apply for extreme usage

## Getting started

    1. Sign up at [venice.ai](https://venice.ai)
    2. Go to **Settings > API Keys > Create new key**
    3. Copy your API key (format: `vapi_xxxxxxxxxxxx`)

    Choose your preferred setup method:

        ```bash
        openclaw onboard --auth-choice venice-api-key
        ```

        This will:
        1. Prompt for your API key (or use existing `VENICE_API_KEY`)
        2. Show all available Venice models
        3. Let you pick your default model
        4. Configure the provider automatically

        ```bash
        export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
        ```

        ```bash
        openclaw onboard --non-interactive \
          --auth-choice venice-api-key \
          --venice-api-key "vapi_xxxxxxxxxxxx"
        ```

    ```bash
    openclaw agent --model venice/kimi-k2-5 --message "Hello, are you working?"
    ```

## Model selection

After setup, OpenClaw shows all available Venice models. Pick based on your needs:

- **Default model**: `venice/kimi-k2-5` for strong private reasoning plus vision.
- **High-capability option**: `venice/claude-opus-4-6` for the strongest anonymized Venice path.
- **Privacy**: Choose "private" models for fully private inference.
- **Capability**: Choose "anonymized" models to access Claude, GPT, Gemini via Venice's proxy.

Change your default model anytime:

```bash
openclaw models set venice/kimi-k2-5
openclaw models set venice/claude-opus-4-6
```

List all available models:

```bash
openclaw models list | grep venice
```

You can also run `openclaw configure`, select **Model/auth**, and choose **Venice AI**.

Use the table below to pick the right model for your use case.

| Use Case                   | Recommended Model                | Why                                          |
| -------------------------- | -------------------------------- | -------------------------------------------- |
| **General chat (default)** | `kimi-k2-5`                      | Strong private reasoning plus vision         |
| **Best overall quality**   | `claude-opus-4-6`                | Strongest anonymized Venice option           |
| **Privacy + coding**       | `qwen3-coder-480b-a35b-instruct` | Private coding model with large context      |
| **Private vision**         | `kimi-k2-5`                      | Vision support without leaving private mode  |
| **Fast + cheap**           | `qwen3-4b`                       | Lightweight reasoning model                  |
| **Complex private tasks**  | `deepseek-v3.2`                  | Strong reasoning, but no Venice tool support |
| **Uncensored**             | `venice-uncensored`              | No content restrictions                      |

## DeepSeek V4 replay behavior

If Venice exposes DeepSeek V4 models such as `venice/deepseek-v4-pro` or
`venice/deepseek-v4-flash`, OpenClaw fills the required DeepSeek V4
`reasoning_content` replay placeholder on assistant messages when the proxy
omits it. Venice rejects DeepSeek's native top-level `thinking` control, so
OpenClaw keeps that provider-specific replay fix separate from the native
DeepSeek provider's thinking controls.

## Built-in catalog (41 total)

    | Model ID                               | Name                                | Context | Features                   |
    | -------------------------------------- | ----------------------------------- | ------- | -------------------------- |
    | `kimi-k2-5`                            | Kimi K2.5                           | 256k    | Default, reasoning, vision |
    | `kimi-k2-thinking`                     | Kimi K2 Thinking                    | 256k    | Reasoning                  |
    | `llama-3.3-70b`                        | Llama 3.3 70B                       | 128k    | General                    |
    | `llama-3.2-3b`                         | Llama 3.2 3B                        | 128k    | General                    |
    | `hermes-3-llama-3.1-405b`              | Hermes 3 Llama 3.1 405B            | 128k    | General, tools disabled    |
    | `qwen3-235b-a22b-thinking-2507`        | Qwen3 235B Thinking                | 128k    | Reasoning                  |
    | `qwen3-235b-a22b-instruct-2507`        | Qwen3 235B Instruct                | 128k    | General                    |
    | `qwen3-coder-480b-a35b-instruct`       | Qwen3 Coder 480B                   | 256k    | Coding                     |
    | `qwen3-coder-480b-a35b-instruct-turbo` | Qwen3 Coder 480B Turbo             | 256k    | Coding                     |
    | `qwen3-5-35b-a3b`                      | Qwen3.5 35B A3B                    | 256k    | Reasoning, vision          |
    | `qwen3-next-80b`                       | Qwen3 Next 80B                     | 256k    | General                    |
    | `qwen3-vl-235b-a22b`                   | Qwen3 VL 235B (Vision)             | 256k    | Vision                     |
    | `qwen3-4b`                             | Venice Small (Qwen3 4B)            | 32k     | Fast, reasoning            |
    | `deepseek-v3.2`                        | DeepSeek V3.2                      | 160k    | Reasoning, tools disabled  |
    | `venice-uncensored`                    | Venice Uncensored (Dolphin-Mistral) | 32k     | Uncensored, tools disabled |
    | `mistral-31-24b`                       | Venice Medium (Mistral)            | 128k    | Vision                     |
    | `google-gemma-3-27b-it`                | Google Gemma 3 27B Instruct        | 198k    | Vision                     |
    | `openai-gpt-oss-120b`                  | OpenAI GPT OSS 120B               | 128k    | General                    |
    | `nvidia-nemotron-3-nano-30b-a3b`       | NVIDIA Nemotron 3 Nano 30B

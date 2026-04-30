---
domain: providers
topic: "Moonshot Kimi Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Moonshot Kimi
  - moonshot provider
  - AI provider
  - model config
  - API key
  - Moonshot
  - Kimi
  - Chinese LLM
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/moonshot.md
---

Moonshot Kimi provider setup, configuration, and model reference for OpenClaw.

Moonshot provides the Kimi API with OpenAI-compatible endpoints. Configure the
provider and set the default model to `moonshot/kimi-k2.6`, or use
Kimi Coding with `kimi/kimi-code`.

Moonshot and Kimi Coding are **separate providers**. Keys are not interchangeable, endpoints differ, and model refs differ (`moonshot/...` vs `kimi/...`).

## Built-in model catalog

[//]: # "moonshot-kimi-k2-ids:start"

| Model ref                         | Name                   | Reasoning | Input       | Context | Max output |
| --------------------------------- | ---------------------- | --------- | ----------- | ------- | ---------- |
| `moonshot/kimi-k2.6`              | Kimi K2.6              | No        | text, image | 262,144 | 262,144    |
| `moonshot/kimi-k2.5`              | Kimi K2.5              | No        | text, image | 262,144 | 262,144    |
| `moonshot/kimi-k2-thinking`       | Kimi K2 Thinking       | Yes       | text        | 262,144 | 262,144    |
| `moonshot/kimi-k2-thinking-turbo` | Kimi K2 Thinking Turbo | Yes       | text        | 262,144 | 262,144    |
| `moonshot/kimi-k2-turbo`          | Kimi K2 Turbo          | No        | text        | 256,000 | 16,384     |

[//]: # "moonshot-kimi-k2-ids:end"

Bundled cost estimates for current Moonshot-hosted K2 models use Moonshot's
published pay-as-you-go rates: Kimi K2.6 is $0.16/MTok cache hit,
$0.95/MTok input, and $4.00/MTok output; Kimi K2.5 is $0.10/MTok cache hit,
$0.60/MTok input, and $3.00/MTok output. Other legacy catalog entries keep
zero-cost placeholders unless you override them in config.

## Getting started

Choose your provider and follow the setup steps.

    **Best for:** Kimi K2 models via the Moonshot Open Platform.

        | Auth choice            | Endpoint                       | Region        |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | International |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | China         |

        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        Or for the China endpoint:

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "moonshot/kimi-k2.6" },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider moonshot
        ```

        Use an isolated state dir when you want to verify model access and cost
        tracking without touching your normal sessions:

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'Reply exactly: KIMI_LIVE_OK' \
          --thinking off \
          --json
        ```

        The JSON response should report `provider: "moonshot"` and
        `model: "kimi-k2.6"`. The assistant transcript entry stores normalized
        token usage plus estimated cost under `usage.cost` when Moonshot returns
        usage metadata.

    ### Config example

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            "moonshot/kimi-k2-thinking": { alias: "Kimi K2 Thinking" },
            "moonshot/kimi-k2-thinking-turbo": { alias: "Kimi K2 Thinking Turbo" },
            "moonshot/kimi-k2-turbo": { alias: "Kimi K2 Turbo" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2-thinking",
                name: "Kimi K2 Thinking",
                reasoning: true,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2-thinking-turbo",
                name: "Kimi K2 Thinking Turbo",
                reasoning: true,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2-turbo",
                name: "Kimi K2 Turbo",
                reasoning: false,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 256000,
                maxTokens: 16384,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

    **Best for:** code-focused tasks via the Kimi Coding endpoint.

    Kimi Coding uses a different API key and provider prefix (`kimi/...`) than Moonshot (`moonshot/...`). Legacy model ref `kimi/k2p5` remains accepted as a compatibility id.

        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-code" },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider kimi
        ```

    ### Config example

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-code" },
          models: {
            "kimi/kimi-code": { alias: "Kimi" },
          },
        },
      },
    }
    ```

## Kimi web search

OpenClaw also ships **Kimi** as a `web_search` provider, backed by Moonshot web
search.

    ```bash
    openclaw configure --section web
    ```

    Choose **Kimi** in the web-search section to store
    `plugins.entries.moonshot.config.webSearch.*`.

    Interactive setup prompts for:

    | Setting             | Options                                                              |
    | ------------------- | -------------------------------------------------------------------- |
    | API region          | `https://api.moonshot.ai/v1` (international) or `https://api.moonshot.cn/v1` (China) |
    | Web search model    | Defaults to `kimi-k2.6`                                             |

Config lives under `plugins.entries.moonshot.config.webSearch`:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## Advanced configuration

    Moonsho

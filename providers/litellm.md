---
domain: providers
topic: "LiteLLM Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - LiteLLM
  - litellm provider
  - AI provider
  - model config
  - API key
  - LiteLLM
  - proxy
  - model proxy
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/litellm.md
---

LiteLLM provider setup, configuration, and model reference for OpenClaw.

[LiteLLM](https://litellm.ai) is an open-source LLM gateway that provides a unified API to 100+ model providers. Route OpenClaw through LiteLLM to get centralized cost tracking, logging, and the flexibility to switch backends without changing your OpenClaw config.

**Why use LiteLLM with OpenClaw?**

- **Cost tracking** — See exactly what OpenClaw spends across all models
- **Model routing** — Switch between Claude, GPT-4, Gemini, Bedrock without config changes
- **Virtual keys** — Create keys with spend limits for OpenClaw
- **Logging** — Full request/response logs for debugging
- **Fallbacks** — Automatic failover if your primary provider is down

## Quick start

    **Best for:** fastest path to a working LiteLLM setup.

        ```bash
        openclaw onboard --auth-choice litellm-api-key
        ```

        For non-interactive setup against a remote proxy, pass the proxy URL explicitly:

        ```bash
        openclaw onboard --non-interactive --auth-choice litellm-api-key --litellm-api-key "$LITELLM_API_KEY" --custom-base-url "https://litellm.example/v1"
        ```

    **Best for:** full control over installation and config.

        ```bash
        pip install 'litellm[proxy]'
        litellm --model claude-opus-4-6
        ```

        ```bash
        export LITELLM_API_KEY="your-litellm-key"

        openclaw
        ```

        That's it. OpenClaw now routes through LiteLLM.

## Configuration

### Environment variables

```bash
export LITELLM_API_KEY="sk-litellm-key"
```

### Config file

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

## Advanced configuration

### Image generation

LiteLLM can also back the `image_generate` tool through OpenAI-compatible
`/images/generations` and `/images/edits` routes. Configure a LiteLLM image
model under `agents.defaults.imageGenerationModel`:

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "litellm/gpt-image-2",
        timeoutMs: 180_000,
      },
    },
  },
}
```

Loopback LiteLLM URLs such as `http://localhost:4000` work without a global
private-network override. For a LAN-hosted proxy, set
`models.providers.litellm.request.allowPrivateNetwork: true` because the API key
will be sent to the configured proxy host.

    Create a dedicated key for OpenClaw with spend limits:

    ```bash
    curl -X POST "http://localhost:4000/key/generate" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "key_alias": "openclaw",
        "max_budget": 50.00,
        "budget_duration": "monthly"
      }'
    ```

    Use the generated key as `LITELLM_API_KEY`.

    LiteLLM can route model requests to different backends. Configure in your LiteLLM `config.yaml`:

    ```yaml
    model_list:
      - model_name: claude-opus-4-6
        litellm_params:
          model: claude-opus-4-6
          api_key: os.environ/ANTHROPIC_API_KEY

      - model_name: gpt-4o
        litellm_params:
          model: gpt-4o
          api_key: os.environ/OPENAI_API_KEY
    ```

    OpenClaw keeps requesting `claude-opus-4-6` — LiteLLM handles the routing.

    Check LiteLLM's dashboard or API:

    ```bash
    # Key info
    curl "http://localhost:4000/key/info" \
      -H "Authorization: Bearer sk-litellm-key"

    # Spend logs
    curl "http://localhost:4000/spend/logs" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY"
    ```

    - LiteLLM runs on `http://localhost:4000` by default
    - OpenClaw connects through LiteLLM's proxy-style OpenAI-compatible `/v1`
      endpoint
    - Native OpenAI-only request shaping does not apply through LiteLLM:
      no `service_tier`, no Responses `store`, no prompt-cache hints, and no
      OpenAI reasoning-compat payload shaping
    - Hidden OpenClaw attribution headers (`originator`, `version`, `User-Agent`)
      are not injected on custom LiteLLM base URLs

For general provider configuration and failover behavior, see [Model Providers](/concepts/model-providers).

## Related

    Official LiteLLM documentation and API reference.

    Overview of all providers, model refs, and failover behavior.

    Full config reference.

    How to choose and configure models.

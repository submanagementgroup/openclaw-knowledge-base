---
domain: gateway
topic: "Local Model Inference Setup"
type: procedure
keywords:
  - local models
  - ollama
  - LM Studio
  - vLLM
  - sgLang
  - local inference
  - self-hosted models
source: ['gateway/local-models.md', 'gateway/local-model-services.md']
related:
  - providers/ollama
  - providers/lmstudio
---

Local models are doable. They also raise the bar on hardware, context size, and prompt-injection defense — small or aggressively quantized cards truncate context and leak safety. This page is the opinionated guide for higher-end local stacks and custom OpenAI-compatible local servers. For lowest-friction onboarding, start with [LM Studio](/providers/lmstudio) or [Ollama](/providers/ollama) and `openclaw onboard`.

For local servers that should start only when a selected model needs them, see
[Local model services](/gateway/local-model-services).

## Hardware floor

Aim high: **≥2 maxed-out Mac Studios or an equivalent GPU rig (~$30k+)** for a comfortable agent loop. A single **24 GB** GPU works only for lighter prompts at higher latency. Always run the **largest / full-size variant you can host**; small or heavily quantized checkpoints raise prompt-injection risk (see [Security](/gateway/security)).

## Pick a backend

| Backend                                              | Use when                                                                    |
| ---------------------------------------------------- | --------------------------------------------------------------------------- |
| [LM Studio](/providers/lmstudio)                     | First-time local setup, GUI loader, native Responses API                    |
| [Ollama](/providers/ollama)                          | CLI workflow, model library, hands-off systemd service                      |
| MLX / vLLM / SGLang                                  | High-throughput self-hosted serving with an OpenAI-compatible HTTP endpoint |
| LiteLLM / OAI-proxy / custom OpenAI-compatible proxy | You front another model API and need OpenClaw to treat it as OpenAI         |

Use Responses API (`api: "openai-responses"`) when the backend supports it (LM Studio does). Otherwise stick to Chat Completions (`api: "openai-completions"`).

> **Note:** **WSL2 + Ollama + NVIDIA/CUDA users:** The official Ollama Linux installer enables a systemd service with `Restart=always`. On WSL2 GPU setups, autostart can reload the last model during boot and pin host memory. If your WSL2 VM repeatedly restarts after enabling Ollama, see [WSL2 crash loop](/providers/ollama#wsl2-crash-loop-repeated-reboots).


## Recommended: LM Studio + large local model (Responses API)

Best current local stack. Load a large model in LM Studio (for example, a full-size Qwen, DeepSeek, or Llama build), enable the local server (default `http://127.0.0.1:1234`), and use Responses API to keep reasoning separate from final text.

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "lmstudio/my-local-model": { alias: "Local" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**Setup checklist**

- Install LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- In LM Studio, download the **largest model build available** (avoid "small"/heavily quantized variants), start the server, confirm `http://127.0.0.1:1234/v1/models` lists it.
- Replace `my-local-model` with the actual model ID shown in LM Studio.
- Keep the model loaded; cold-load adds startup latency.
- Adjust `contextWindow`/`maxTokens` if your LM Studio build differs.
- For WhatsApp, stick to Responses API so only final text is sent.

Keep hosted models configured even when running local; use `models.mode: "merge"` so fallbacks stay available.

### Hybrid config: hosted primary, local fallback

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Local-first with hosted safety net

Swap the primary and fallback order; keep the same providers block and `models.mode: "merge"` so you can fall back to Sonnet or Opus when the local box is down.

### Regional hosting / data routing

- Hosted MiniMax/Kimi/GLM variants also exist on OpenRouter with region-pinned endpoints (e.g., US-hosted). Pick the regional variant there to keep traffic in your chosen jurisdiction while still using `models.mode: "merge"` for Anthropic/OpenAI fallbacks.
- Local-only remains the strongest privacy path; hosted regional routing is the middle ground when you need provider features but want control over data flow.

## Other OpenAI-compatible local proxies

MLX (`mlx_lm.server`), vLLM, SGLang, LiteLLM, OAI-proxy, or custom
gateways work if they expose an OpenAI-style `/v1/chat/completions`
endpoint. Use the Chat Completions adapter unless the backend explicitly
documents `/v1/responses` support. Replace the provider block above with your
endpoint and model ID:

```json5
{
  agents: {
    defaults: {
      model: { primary: "local/my-local-model" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

If `api` is omitted on a custom provider with a `baseUrl`, OpenClaw defaults to
`openai-completions`. Loopback endpoints such as `127.0.0.1` are trusted
automatically; LAN, tailnet, and private DNS endpoints still need
`request.allowPrivateNetwork: true`.

The `models.providers.<id>.models[].id` value is provider-local. Do not
include the provider prefix there. For example, an MLX server started with
`mlx_lm.server --model mlx-community/Qwen3-30B-A3B-6bit` should use this
catalog id and model ref:

- `models.providers.mlx.models[].id: "mlx-community/Qwen3-30B-A3B-6bit"`
- `agents.defaults.model.primary: "mlx/mlx-community/Qwen3-30B-A3B-6bit"`

Set `input: ["text", "image"]` on local or proxied vision models so image
attachments are injected into agent turns. Interactive custom-provider
onboarding infers common vision model IDs and asks only for unknown names.
Non-interactive onboarding uses the same inference; use `--custom-image-input`
for unknown vision IDs or `--custom-text-input` when a known-looking model is
text-only behind your endpoint.

Keep `models.mode: "merge"` so hosted models stay available as fallbacks.
Use `models.providers.<id>.timeoutSeconds` for slow local or remote model
servers before raising `agents.defaults.timeoutSeconds`. The provider timeout
applies only to model HTTP requests, including connect, headers, body streaming,
and the total guarded-fetch abort.

> **Note:*

## Local Model Services

`models.providers.<id>.localService` lets OpenClaw start a provider-owned local
model server on demand. It is provider-level config: when the selected model
belongs to that provider, OpenClaw probes the service, starts the process if the
endpoint is down, waits for readiness, then sends the model request.

Use it for local servers that are expensive to keep running all day, or for
manual setups where model selection should be enough to bring the backend up.

## How it works

1. A model request resolves to a configured provider.
2. If that provider has `localService`, OpenClaw probes `healthUrl`.
3. If the probe succeeds, OpenClaw uses the existing server.
4. If the probe fails, OpenClaw starts `command` with `args`.
5. OpenClaw polls readiness until `readyTimeoutMs` expires.
6. The model request is sent through the normal provider transport.
7. If OpenClaw started the process and `idleStopMs` is positive, the process is
   stopped after the last in-flight request has been idle for that long.

OpenClaw does not install launchd, systemd, Docker, or a daemon for this. The
server is a child process of the OpenClaw process that first needed it.

## Config shape

```json5
{
  models: {
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "local-model",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/absolute/path/to/server",
          args: ["--host", "127.0.0.1", "--port", "8000"],
          cwd: "/absolute/path/to/working-dir",
          env: { LOCAL_MODEL_CACHE: "/absolute/path/to/cache" },
          healthUrl: "http://127.0.0.1:8000/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "my-local-model",
            name: "My Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

## Fields

- `command`: absolute executable path. Shell lookup is not used.
- `args`: process arguments. No shell expansion, pipes, globbing, or quoting
  rules are applied.
- `cwd`: optional working directory for the process.
- `env`: optional environment variables merged over the OpenClaw process
  environment.
- `healthUrl`: readiness URL. If omitted, OpenClaw appends `/models` to
  `baseUrl`, so `http://127.0.0.1:8000/v1` becomes
  `http://127.0.0.1:8000/v1/models`.
- `readyTimeoutMs`: startup readiness deadline. Default: `120000`.
- `idleStopMs`: idle shutdown delay for OpenClaw-started processes. `0` or
  omitted keeps the process alive until OpenClaw exits.

## Inferrs example

Inferrs is a custom OpenAI-compatible `/v1` backend, so the same local service
API works with the `inferrs` provider entry.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

Replace `command` with the result of `which inferrs` on the machine running
OpenClaw.

## ds4 example

```json5
{
  models: {
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/Users/you/Projects/oss/ds4/ds4-server",
          args: [
            "--model",
            "/Users/you/Projects/oss/ds4/ds4flash.gguf",
            "--host",
            "127.0.0.1",
            "--port",
            "18000",
            "--ctx",
            "393216",
          ],
          cwd: "/Users/you/Projects/oss/ds4",
          healthUrl: "http://127.0.0.1:18000/v1/models",
          readyTimeoutMs: 300000,
          idleStopMs: 0,
        },
        models: [],
      },
    },
  },
}
```

## Operational notes

- One OpenClaw process manages the child it started. Another OpenClaw process
  that sees the same health URL already live will reuse it without adopting it.
- Startup is serialized per provider command and argument set, so concurrent
  requests do not spawn duplicate servers for the same config.
- Active streaming responses hold a lease; idle shutdown waits until response
  body handling is complete.
- Use `timeoutSeconds` on slow local providers so cold starts and long generations
  do not hit the default model request timeout.
- Use an explicit `healthUrl` if your server exposes readiness somewhere other
  than `/v1/models`.

## Related

**Local models:** Local model setup, provider choices, and safety guidance.
  

**Inferrs:** Run OpenClaw through the inferrs OpenAI-compatible local server.
  


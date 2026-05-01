---
domain: gateway
topic: "Local AI Models: Ollama, LM Studio, vLLM, and SGLang Configuration"
type: procedure
keywords:
  - local models
  - Ollama
  - LM Studio
  - vLLM
  - SGLang
  - self-hosted models
  - offline models
related:
  - providers/ollama
  - providers/lmstudio
  - providers/vllm
  - gateway/configuration-overview
source: gateway/local-models.md
---

Running AI models locally with OpenClaw via Ollama, LM Studio, vLLM, or SGLang. Local models run on your machine and require no API key.

## Ollama (Recommended for Local Models)

```bash
# Install Ollama
brew install ollama  # macOS
curl -fsSL https://ollama.ai/install.sh | sh  # Linux

# Pull a model
ollama pull llama3.2

# Configure OpenClaw to use it
```

```json5
{
  agents: {
    defaults: {
      model: "ollama/llama3.2",
    }
  },
  models: {
    providers: {
      ollama: {
        baseUrl: "http://localhost:11434",
      }
    }
  }
}
```

## LM Studio

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
      }
    }
  }
}
```

## Implicit Local Provider Discovery

OpenClaw auto-discovers Ollama and LM Studio if they are running on standard ports. Run `openclaw models list` to see discovered local models.

Local is doable, but OpenClaw expects large context + strong defenses against prompt injection. Small cards truncate context and leak safety. Aim high: **≥2 maxed-out Mac Studios or equivalent GPU rig (~$30k+)**. A single **24 GB** GPU works only for lighter prompts with higher latency. Use the **largest / full-size model variant you can run**; aggressively quantized or “small” checkpoints raise prompt-injection risk (see [Security](/gateway/security)).

If you want the lowest-friction local setup, start with [LM Studio](/providers/lmstudio) or [Ollama](/providers/ollama) and `openclaw onboard`. This page is the opinionated guide for higher-end local stacks and custom OpenAI-compatible local servers.

**WSL2 + Ollama + NVIDIA/CUDA users:** The official Ollama Linux installer enables a systemd service with `Restart=always`. On WSL2 GPU setups, autostart can reload the last model during boot and pin host memory. If your WSL2 VM repeatedly restarts after enabling Ollama, see [WSL2 crash loop](/providers/ollama#wsl2-crash-loop-repeated-reboots).

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
- In LM Studio, download the **largest model build available** (avoid “small”/heavily quantized variants), start the server, confirm `http://127.0.0.1:1234/v1/models` lists it.
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

- Hosted MiniMax/Kimi/GLM variants a

## Context Window Preflight Thresholds

OpenClaw derives context-window preflight thresholds from the detected model window, or from the uncapped model window when `agents.defaults.contextTokens` lowers the effective window. Warnings trigger below 20% with an **8k** floor. Hard blocks use the 10% threshold with a **4k** floor, capped to the effective context window so oversized model metadata cannot reject an otherwise valid user cap.

If you hit a context preflight block, raise the server/model context limit or choose a larger model. For memory pressure: lower `contextWindow` or raise your server limit.

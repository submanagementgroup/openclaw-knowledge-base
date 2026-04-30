---
domain: providers
topic: "Z.AI: GLM Model Provider Setup and Configuration"
type: integration
keywords:
  - Z.AI
  - ZAI_API_KEY
  - GLM models
  - zai provider
  - glm-5.1
  - glm-4.7
  - glm-4.5
  - openclaw zai
  - zai-api-key onboarding
  - tool_stream zai
  - zai thinking
  - zai image understanding
source: providers/zai.md
related:
  - providers/providers-overview
  - concepts/models
---

Z.AI is the API platform for **GLM** models, using Bearer auth with an API key. OpenClaw uses the `zai` provider. The default bundled model ref is `zai/glm-5.1`.

## Getting Started

### Auto-Detect Endpoint (Recommended)

```bash
openclaw onboard --auth-choice zai-api-key
```

Then set a default model in `~/.openclaw/openclaw.json`:

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5.1" } } },
}
```

Verify the model is available:

```bash
openclaw models list --provider zai
```

### Explicit Regional Endpoint

```bash
# Coding Plan Global (recommended for Coding Plan users)
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (China region)
openclaw onboard --auth-choice zai-coding-cn

# General API
openclaw onboard --auth-choice zai-global

# General API CN (China region)
openclaw onboard --auth-choice zai-cn
```

## Built-In Model Catalog

| Model ref | Notes |
|-----------|-------|
| `zai/glm-5.1` | Default model |
| `zai/glm-5` | |
| `zai/glm-5-turbo` | |
| `zai/glm-5v-turbo` | |
| `zai/glm-4.7` | |
| `zai/glm-4.7-flash` | |
| `zai/glm-4.7-flashx` | |
| `zai/glm-4.6` | |
| `zai/glm-4.6v` | |
| `zai/glm-4.5` | |
| `zai/glm-4.5-air` | |
| `zai/glm-4.5-flash` | |
| `zai/glm-4.5v` | |

Use model refs as `zai/<model>` (example: `zai/glm-5`). Unknown `glm-5*` ids still forward-resolve on the bundled provider path by synthesizing metadata from the `glm-4.7` template when the id matches the GLM-5 family shape.

## Advanced Configuration

### Tool-Call Streaming

`tool_stream` is enabled by default for Z.AI tool-call streaming. To disable:

```json5
{
  agents: {
    defaults: {
      models: {
        "zai/<model>": {
          params: { tool_stream: false },
        },
      },
    },
  },
}
```

### Thinking

Z.AI thinking follows OpenClaw's `/think` controls. With thinking off, OpenClaw sends `thinking: { type: "disabled" }` to avoid spending the output budget on `reasoning_content` before visible text.

**Preserved thinking** is opt-in (Z.AI requires full historical `reasoning_content` to be replayed, increasing prompt tokens):

```json5
{
  agents: {
    defaults: {
      models: {
        "zai/glm-5.1": {
          params: { preserveThinking: true },
        },
      },
    },
  },
}
```

When enabled and thinking is on, OpenClaw sends `thinking: { type: "enabled", clear_thinking: false }` and replays prior `reasoning_content`.

Override the exact provider payload with `params.extra_body.thinking` for advanced use.

### Image Understanding

The bundled Z.AI plugin registers image understanding using `glm-4.6v`. Image understanding is auto-resolved from the configured Z.AI auth — no additional config needed.

### Auth Details

- Z.AI uses Bearer auth with your API key.
- The `zai-api-key` onboarding choice auto-detects the matching Z.AI endpoint from the key prefix.
- Use explicit regional choices (`zai-coding-global`, `zai-coding-cn`, `zai-global`, `zai-cn`) when you want to force a specific API surface.

## Related

- [Providers overview](/providers/providers-overview)
- [Model selection](/concepts/models)

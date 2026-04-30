---
domain: providers
topic: "OpenCode Go Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - OpenCode Go
  - opencode-go provider
  - AI provider
  - model config
  - API key
  - OpenCode Go
  - coding agent
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/opencode-go.md
---

OpenCode Go provider setup, configuration, and model reference for OpenClaw.

OpenCode Go is the Go catalog within [OpenCode](/providers/opencode).
It uses the same `OPENCODE_API_KEY` as the Zen catalog, but keeps the runtime
provider id `opencode-go` so upstream per-model routing stays correct.

| Property         | Value                           |
| ---------------- | ------------------------------- |
| Runtime provider | `opencode-go`                   |
| Auth             | `OPENCODE_API_KEY`              |
| Parent setup     | [OpenCode](/providers/opencode) |

## Built-in catalog

OpenClaw sources most Go catalog rows from the bundled pi model registry and
supplements current upstream rows while the registry catches up. Run
`openclaw models list --provider opencode-go` for the current model list.

The provider includes:

| Model ref                       | Name                  |
| ------------------------------- | --------------------- |
| `opencode-go/glm-5`             | GLM-5                 |
| `opencode-go/glm-5.1`           | GLM-5.1               |
| `opencode-go/kimi-k2.5`         | Kimi K2.5             |
| `opencode-go/kimi-k2.6`         | Kimi K2.6 (3x limits) |
| `opencode-go/deepseek-v4-pro`   | DeepSeek V4 Pro       |
| `opencode-go/deepseek-v4-flash` | DeepSeek V4 Flash     |
| `opencode-go/mimo-v2-omni`      | MiMo V2 Omni          |
| `opencode-go/mimo-v2-pro`       | MiMo V2 Pro           |
| `opencode-go/minimax-m2.5`      | MiniMax M2.5          |
| `opencode-go/minimax-m2.7`      | MiniMax M2.7          |
| `opencode-go/qwen3.5-plus`      | Qwen3.5 Plus          |
| `opencode-go/qwen3.6-plus`      | Qwen3.6 Plus          |

## Getting started

        ```bash
        openclaw onboard --auth-choice opencode-go
        ```

        ```bash
        openclaw config set agents.defaults.model.primary "opencode-go/kimi-k2.6"
        ```

        ```bash
        openclaw models list --provider opencode-go
        ```

        ```bash
        openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
        ```

        ```bash
        openclaw models list --provider opencode-go
        ```

## Config example

```json5
{
  env: { OPENCODE_API_KEY: "YOUR_API_KEY_HERE" }, // pragma: allowlist secret
  agents: { defaults: { model: { primary: "opencode-go/kimi-k2.6" } } },
}
```

## Advanced configuration

    OpenClaw handles per-model routing automatically when the model ref uses
    `opencode-go/...`. No additional provider config is required.

    Runtime refs stay explicit: `opencode/...` for Zen, `opencode-go/...` for Go.
    This keeps upstream per-model routing correct across both catalogs.

    The same `OPENCODE_API_KEY` is used by both the Zen and Go catalogs. Entering
    the key during setup stores credentials for both runtime providers.

See [OpenCode](/providers/opencode) for the shared onboarding overview and the full
Zen + Go catalog reference.

## Related

    Shared onboarding, catalog overview, and advanced notes.

    Choosing providers, model refs, and failover behavior.

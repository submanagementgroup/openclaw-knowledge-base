---
domain: providers
topic: "OpenCode Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - OpenCode
  - opencode provider
  - AI provider
  - model config
  - API key
  - OpenCode
  - coding agent
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/opencode.md
---

OpenCode provider setup, configuration, and model reference for OpenClaw.

OpenCode exposes two hosted catalogs in OpenClaw:

| Catalog | Prefix            | Runtime provider |
| ------- | ----------------- | ---------------- |
| **Zen** | `opencode/...`    | `opencode`       |
| **Go**  | `opencode-go/...` | `opencode-go`    |

Both catalogs use the same OpenCode API key. OpenClaw keeps the runtime provider ids
split so upstream per-model routing stays correct, but onboarding and docs treat them
as one OpenCode setup.

## Getting started

    **Best for:** the curated OpenCode multi-model proxy (Claude, GPT, Gemini).

        ```bash
        openclaw onboard --auth-choice opencode-zen
        ```

        Or pass the key directly:

        ```bash
        openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
        ```

        ```bash
        openclaw config set agents.defaults.model.primary "opencode/claude-opus-4-6"
        ```

        ```bash
        openclaw models list --provider opencode
        ```

    **Best for:** the OpenCode-hosted Kimi, GLM, and MiniMax lineup.

        ```bash
        openclaw onboard --auth-choice opencode-go
        ```

        Or pass the key directly:

        ```bash
        openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
        ```

        ```bash
        openclaw config set agents.defaults.model.primary "opencode-go/kimi-k2.6"
        ```

        ```bash
        openclaw models list --provider opencode-go
        ```

## Config example

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## Built-in catalogs

### Zen

| Property         | Value                                                                   |
| ---------------- | ----------------------------------------------------------------------- |
| Runtime provider | `opencode`                                                              |
| Example models   | `opencode/claude-opus-4-6`, `opencode/gpt-5.5`, `opencode/gemini-3-pro` |

### Go

| Property         | Value                                                                    |
| ---------------- | ------------------------------------------------------------------------ |
| Runtime provider | `opencode-go`                                                            |
| Example models   | `opencode-go/kimi-k2.6`, `opencode-go/glm-5`, `opencode-go/minimax-m2.5` |

## Advanced configuration

    `OPENCODE_ZEN_API_KEY` is also supported as an alias for `OPENCODE_API_KEY`.

    Entering one OpenCode key during setup stores credentials for both runtime
    providers. You do not need to onboard each catalog separately.

    You sign in to OpenCode, add billing details, and copy your API key. Billing
    and catalog availability are managed from the OpenCode dashboard.

    Gemini-backed OpenCode refs stay on the proxy-Gemini path, so OpenClaw keeps
    Gemini thought-signature sanitation there without enabling native Gemini
    replay validation or bootstrap rewrites.

    Non-Gemini OpenCode refs keep the minimal OpenAI-compatible replay policy.

Entering one OpenCode key during setup stores credentials for both the Zen and
Go runtime providers, so you only need to onboard once.

## Related

    Choosing providers, model refs, and failover behavior.

    Full config reference for agents, models, and providers.

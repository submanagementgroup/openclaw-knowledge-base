---
domain: providers
topic: "Anthropic Claude Provider: Claude Opus, Sonnet, Haiku, OAuth, and Prompt Caching"
type: procedure
keywords:
  - Anthropic
  - Claude
  - claude-opus-4-5
  - ANTHROPIC_API_KEY
  - Claude Max
  - OAuth
  - prompt caching
related:
  - providers/openai
  - providers/google
  - concepts/oauth-auth-profiles
source: providers/anthropic.md
---

Anthropic Claude provider for OpenClaw. Supports Claude Opus, Sonnet, and Haiku models with tool use, streaming, prompt caching, and OAuth.

## Quick Setup

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-5",
    }
  }
}
```

Set `ANTHROPIC_API_KEY` in your environment. Or use Claude Max OAuth (see below).

Anthropic builds the **Claude** model family. OpenClaw supports two auth routes:

- **API key** — direct Anthropic API access with usage-based billing (`anthropic/*` models)
- **Claude CLI** — reuse an existing Claude CLI login on the same host

Anthropic staff told us OpenClaw-style Claude CLI usage is allowed again, so
OpenClaw treats Claude CLI reuse and `claude -p` usage as sanctioned unless
Anthropic publishes a new policy.

For long-lived gateway hosts, Anthropic API keys are still the clearest and
most predictable production path.

Anthropic's current public docs:

- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Using Claude Code with your Team or Enterprise plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

## Getting started

    **Best for:** standard API access and usage-based billing.

        Create an API key in the [Anthropic Console](https://console.anthropic.com/).

        ```bash
        openclaw onboard
        # choose: Anthropic API key
        ```

        Or pass the key directly:

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```

        ```bash
        openclaw models list --provider anthropic
        ```

    ### Config example

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "sk-ant-..." },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
    }
    ```

    **Best for:** reusing an existing Claude CLI login without a separate API key.

        Verify with:

        ```bash
        claude --version
        ```

        ```bash
        openclaw onboard
        # choose: Claude CLI
        ```

        OpenClaw detects and reuses the existing Claude CLI credentials.

        ```bash
        openclaw models list --provider anthropic
        ```

    Setup and runtime details for the Claude CLI backend are in [CLI Backends](/gateway/cli-backends).

    ### Config example

    Prefer the canonical Anthropic model ref plus a CLI runtime override:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-7" },
          agentRuntime: { id: "claude-cli" },
        },
      },
    }
    ```

    Legacy `claude-cli/claude-opus-4-7` model refs still work for
    compatibility, but new config should keep provider/model selection as
    `anthropic/*` and put the execution backend in `agentRuntime.id`.

    If you want the clearest billing path, use an Anthropic API key instead. OpenClaw also supports subscription-style options from [OpenAI Codex](/providers/openai), [Qwen Cloud](/providers/qwen), [MiniMax](/providers/minimax), and [Z.AI / GLM](/providers/glm).

## Thinking defaults (Claude 4.6)

Claude 4.6 models default to `adaptive` thinking in OpenClaw when no explicit thinking level is set.

Override per-message with `/think:<level>` or in model params:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { thinking: "adaptive" },
        },
      },
    },
  },
}
```

Related Anthropic docs:
- [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## Prompt caching

OpenClaw supports Anthropic's prompt caching feature for API-key auth.

| Value               | Cache duration | Description                            |
| ------------------- | -------------- | -------------------------------------- |
| `"short"` (default) | 5 minutes      | Applied automatically for API-key auth |
| `"long"`            | 1 hour         | Extended cache                         |
| `"none"`            | No caching     | Disable prompt caching                 |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

    Use model-level params as your baseline, then override specific agents via `agents.list[].params`:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    Config merge order:

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.list[].params` (matching `id`, overrides by key)

    This lets one agent keep a long-lived cache while another agent on the same model disables caching for bursty/low-reuse traffic.

    - Anthropic Claude models on Bedrock (`amazon-bedrock/*anthropic.claude*`) accept `cacheRetention` pass-through when configured.
    - Non-Anthropic Bedrock models are forced to `cacheRetention: "none"` at runtime.
    - API-key smart defaults also seed `cacheRetention: "short"` for Claude-on-Bedrock refs when no explicit value is set.

## Advanced configuration

    OpenClaw's shared `/fast` toggle supports direct Anthropic traffic (API-key and OAuth to `api.anthropic.com`).

    | Command | Maps to |
    |---------|---------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    - Only injected for direct `api.anthropic.com` requests. Proxy routes leave `service_tier` untouched.
    - Explicit `serviceTier` or `service_tier` params override `/fast` when both are set.
    - On accounts without Priority Tier capacity, `service_tier: "auto"` may resolve to `standard`.

    The bundled Anthropic plugin registers image and PDF understanding. OpenClaw
    auto-resolves media capabilities from the configured Anthropic auth — no
    additional config is needed.

    | Property       | Value                |
    | -------------- | -------------------- |
    | Default model  | `claude-opus-4-6`    |
    | Supported input | Images, PDF documents |

    When an image or PDF is attached to a conversation, OpenClaw automatically
    routes it through the Anthropic media understanding provider.

    Anthropic's 1M context window is beta-gated. Enable it per model:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-4-6": {
              params: { context1m: true },
            },
          },

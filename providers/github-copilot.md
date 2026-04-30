---
domain: providers
topic: "GitHub Copilot Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - GitHub Copilot
  - github-copilot provider
  - AI provider
  - model config
  - API key
  - GitHub Copilot
  - Copilot
  - GitHub models
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/github-copilot.md
---

GitHub Copilot provider setup, configuration, and model reference for OpenClaw.

GitHub Copilot is GitHub's AI coding assistant. It provides access to Copilot
models for your GitHub account and plan. OpenClaw can use Copilot as a model
provider in two different ways.

## Two ways to use Copilot in OpenClaw

    Use the native device-login flow to obtain a GitHub token, then exchange it for
    Copilot API tokens when OpenClaw runs. This is the **default** and simplest path
    because it does not require VS Code.

        ```bash
        openclaw models auth login-github-copilot
        ```

        You will be prompted to visit a URL and enter a one-time code. Keep the
        terminal open until it completes.

        ```bash
        openclaw models set github-copilot/claude-opus-4.7
        ```

        Or in config:

        ```json5
        {
          agents: {
            defaults: { model: { primary: "github-copilot/claude-opus-4.7" } },
          },
        }
        ```

    Use the **Copilot Proxy** VS Code extension as a local bridge. OpenClaw talks to
    the proxy's `/v1` endpoint and uses the model list you configure there.

    Choose this when you already run Copilot Proxy in VS Code or need to route
    through it. You must enable the plugin and keep the VS Code extension running.

## Optional flags

| Flag            | Description                                         |
| --------------- | --------------------------------------------------- |
| `--yes`         | Skip the confirmation prompt                        |
| `--set-default` | Also apply the provider's recommended default model |

```bash
# Skip confirmation
openclaw models auth login-github-copilot --yes

# Login and set the default model in one step
openclaw models auth login --provider github-copilot --method device --set-default
```

## Non-interactive onboarding

If you already have a GitHub OAuth access token for Copilot, import it during
headless setup with `openclaw onboard --non-interactive`:

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice github-copilot \
  --github-copilot-token "$COPILOT_GITHUB_TOKEN" \
  --skip-channels --skip-health
```

You can also omit `--auth-choice`; passing `--github-copilot-token` infers the
GitHub Copilot provider auth choice. If the flag is omitted, onboarding falls
back to `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, then `GITHUB_TOKEN`. Use
`--secret-input-mode ref` with `COPILOT_GITHUB_TOKEN` set to store an env-backed
`tokenRef` instead of plaintext in `auth-profiles.json`.

    The device-login flow requires an interactive TTY. Run it directly in a
    terminal, not in a non-interactive script or CI pipeline.

    Copilot model availability depends on your GitHub plan. If a model is
    rejected, try another ID (for example `github-copilot/gpt-4.1`).

    Claude model IDs use the Anthropic Messages transport automatically. GPT,
    o-series, and Gemini models keep the OpenAI Responses transport. OpenClaw
    selects the correct transport based on the model ref.

    OpenClaw sends Copilot IDE-style request headers on Copilot transports,
    including built-in compaction, tool-result, and image follow-up turns. It
    does not enable provider-level Responses continuation for Copilot unless
    that behavior has been verified against Copilot's API.

    OpenClaw resolves Copilot auth from environment variables in the following
    priority order:

    | Priority | Variable              | Notes                            |
    | -------- | --------------------- | -------------------------------- |
    | 1        | `COPILOT_GITHUB_TOKEN` | Highest priority, Copilot-specific |
    | 2        | `GH_TOKEN`            | GitHub CLI token (fallback)      |
    | 3        | `GITHUB_TOKEN`        | Standard GitHub token (lowest)   |

    When multiple variables are set, OpenClaw uses the highest-priority one.
    The device-login flow (`openclaw models auth login-github-copilot`) stores
    its token in the auth profile store and takes precedence over all environment
    variables.

    The login stores a GitHub token in the auth profile store and exchanges it
    for a Copilot API token when OpenClaw runs. You do not need to manage the
    token manually.

The device-login command requires an interactive TTY. Use non-interactive
onboarding when you need headless setup.

## Memory search embeddings

GitHub Copilot can also serve as an embedding provider for
[memory search](/concepts/memory-search). If you have a Copilot subscription and
have logged in, OpenClaw can use it for embeddings without a separate API key.

### Auto-detection

When `memorySearch.provider` is `"auto"` (the default), GitHub Copilot is tried
at priority 15 -- after local embeddings but before OpenAI and other paid
providers. If a GitHub token is available, OpenClaw discovers available
embedding models from the Copilot API and picks the best one automatically.

### Explicit config

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "github-copilot",
        // Optional: override the auto-discovered model
        model: "text-embedding-3-small",
      },
    },
  },
}
```

### How it works

1. OpenClaw resolves your GitHub token (from env vars or auth profile).
2. Exchanges it for a short-lived Copilot API token.
3. Queries the Copilot `/models` endpoint to discover available embedding models.
4. Picks the best model (prefers `text-embedding-3-small`).
5. Sends embedding requests to the Copilot `/embeddings` endpoint.

Model availability depends on your GitHub plan. If no embedding models are
available, OpenClaw skips Copilot and tries the next provider.

## Related

    Choosing providers, model refs, and failover behavior.

    Auth details and credential reuse rules.

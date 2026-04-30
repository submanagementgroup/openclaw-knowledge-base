---
domain: providers
topic: "OpenAI Advanced: Azure OpenAI Endpoints, Codex OAuth, and Voice Configuration"
type: reference
keywords:
  - Azure OpenAI
  - OpenAI Azure
  - Codex OAuth
  - OpenAI voice
  - TTS OpenAI
  - OpenAI embeddings
related:
  - providers/openai
  - providers/anthropic
source: providers/openai.md
---

OpenAI Azure endpoints, advanced configuration, and Codex harness setup.

enai/<model>` model provider                            | Yes                                                    |
| Codex subscription models | `openai-codex/<model>` with `openai-codex` OAuth           | Yes                                                    |
| Codex app-server harness  | `openai/<model>` with `agentRuntime.id: codex`             | Yes                                                    |
| Server-side web search    | Native OpenAI Responses tool                               | Yes, when web search is enabled and no provider pinned |
| Images                    | `image_generate`                                           | Yes                                                    |
| Videos                    | `video_generate`                                           | Yes                                                    |
| Text-to-speech            | `messages.tts.provider: "openai"` / `tts`                  | Yes                                                    |
| Batch speech-to-text      | `tools.media.audio` / media understanding                  | Yes                                                    |
| Streaming speech-to-text  | Voice Call `streaming.provider: "openai"`                  | Yes                                                    |
| Realtime voice            | Voice Call `realtime.provider: "openai"` / Control UI Talk | Yes                                                    |
| Embeddings                | memory embedding provider                                  | Yes                                                    |

## Memory embeddings

OpenClaw can use OpenAI, or an OpenAI-compatible embedding endpoint, for
`memory_search` indexing and query embeddings:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
      },
    },
  },
}
```

For OpenAI-compatible endpoints that require asymmetric embedding labels, set
`queryInputType` and `documentInputType` under `memorySearch`. OpenClaw forwards
those as provider-specific `input_type` request fields: query embeddings use
`queryInputType`; indexed memory chunks and batch indexing use
`documentInputType`. See the [Memory configuration reference](/reference/memory-config#provider-specific-config) for the full example.

## Getting started

Choose your preferred auth method and follow the setup steps.

    **Best for:** direct API access and usage-based billing.

        Create or copy an API key from the [OpenAI Platform dashboard](https://platform.openai.com/api-keys).

        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        Or pass the key directly:

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```

        ```bash
        openclaw models list --provider openai
        ```

    ### Route summary

    | Model ref              | Runtime config             | Route                       | Auth             |
    | ---------------------- | -------------------------- | --------------------------- | ---------------- |
    | `openai/gpt-5.5`       | omitted / `agentRuntime.id: "pi"`    | Direct OpenAI Platform API  | `OPENAI_API_KEY` |
    | `openai/gpt-5.4-mini`  | omitted / `agentRuntime.id: "pi"`    | Direct OpenAI Platform API  | `OPENAI_API_KEY` |
    | `openai/gpt-5.5`       | `agentRuntime.id: "codex"`           | Codex app-server harness    | Codex app-server |

    `openai/*` is the direct OpenAI API-key route unless you explicitly force
    the Codex app-server harness. Use `openai-codex/*` for Codex OAuth through
    the default PI runner, or use `openai/gpt-5.5` with
    `agentRuntime.id: "codex"` for native Codex app-server execution.

    ### Config example

    ```json5
    {
      env: { OPENAI_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "openai/gpt-5.5" } } },
    }
    ```

    OpenClaw does **not** expose `openai/gpt-5.3-codex-spark`. Live OpenAI API requests reject that model, and the current Codex catalog does not expose it either.

    **Best for:** using your ChatGPT/Codex subscription instead of a separate API key. Codex cloud requires ChatGPT sign-in.

        ```bash
        openclaw onboard --auth-choice openai-codex
        ```

        Or run OAuth directly:

        ```bash
        openclaw models auth login --provider openai-codex
        ```

        For headless or callback-hostile setups, add `--device-code` to sign in with a ChatGPT device-code flow instead of the localhost browser callback:

        ```bash
        openclaw models auth login --provider openai-codex --device-code
        ```

        ```bash
        openclaw config set agents.defaults.model.primary openai-codex/gpt-5.5
        ```

        ```bash
        openclaw models list --provider openai-codex
        ```

    ### Route summary

    | Model ref | Runtime config | Route | Auth |
    |-----------|----------------|-------|------|
    | `openai-codex/gpt-5.5` | omitted / `runtime: "pi"` | ChatGPT/Codex OAuth through PI | Codex sign-in |
    | `openai-codex/gpt-5.5` | `runtime: "auto"` | Still PI unless a plugin explicitly claims `openai-codex` | Codex sign-in |
    | `openai/gpt-5.5` | `agentRuntime.id: "codex"` | Codex app-server harness | Codex app-server auth |

    Keep using the `openai-codex` provider id for auth/profile commands. The
    `openai-codex/*` model prefix is also the explicit PI route for Codex OAuth.
    It does not select or auto-enable the bundled Codex app-server harness.

    `openai-codex/gpt-5.4-mini` is not a supported Codex OAuth route. Use
    `openai/gpt-5.4-mini` with an OpenAI API key, or use
    `openai-codex/gpt-5.5` with Codex OAuth.

    ### Config example

    ```json5
    {
      agents: { defaults: { model: { primary: "openai-codex/gpt-5.5" } } },
    }
    ```

    Onboarding no longer imports OAuth material from `~/.codex`. Sign in with browser OAuth (default) or the device-code flow above — OpenClaw manages the resulting credentials in its own agent auth store.

    ### Status indicator

    Chat `/status` shows which model runtime is active for the current session.
    The default PI harness appears as `Runtime: OpenClaw Pi Default`. When the
    bundled Codex app-server harness is selected, `/status` shows
    `Runtime: OpenAI Codex`. Existing sessions keep their recorded harness id, so use
    `/new` or `/reset` after changing `agentRuntime` if you want `/status` to
    reflect a new PI/Codex choice.

    ### Doctor warning

    If the bundled `codex` plugin is enabled while this tab's
    `openai-codex/*` route is selected, `openclaw doctor` warns that the model
    still resolves through PI. Keep the config unchanged when that is the
    intended subscription-auth route. Switch to `openai/<model>` plus
    `agentRuntime.id: "codex"` only when you want native Codex
    app-server execution.

    ### Context window cap

    OpenClaw treats model metadata and the

---
domain: providers
topic: "OpenAI Provider"
type: integration
keywords:
  - openai
  - gpt-4o
  - ChatGPT
  - OpenAI API
  - OPENAI_API_KEY
  - openai provider config
  - dall-e
source: providers/openai.md
---

OpenAI provides developer APIs for GPT models, and Codex is also available as a
ChatGPT-plan coding agent through OpenAI's Codex clients. OpenClaw keeps those
surfaces separate so config stays predictable.

OpenClaw uses `openai/*` as the canonical OpenAI model route. Embedded agent
turns on OpenAI models run through the native Codex app-server runtime by
default; direct OpenAI API-key auth remains available for non-agent OpenAI
surfaces such as images, embeddings, speech, and realtime.

- **Agent models** - `openai/*` models through the Codex runtime; sign in with
  Codex auth for ChatGPT/Codex subscription use, or configure a Codex-compatible
  OpenAI API-key backup when you intentionally want API-key auth.
- **Non-agent OpenAI APIs** - direct OpenAI Platform access with usage-based
  billing through `OPENAI_API_KEY` or OpenAI API-key onboarding.
- **Legacy config** - `openai-codex/*` model refs are repaired by
  `openclaw doctor --fix` to `openai/*` plus the Codex runtime.

OpenAI explicitly supports subscription OAuth usage in external tools and workflows like OpenClaw.

Provider, model, runtime, and channel are separate layers. If those labels are
getting mixed together, read [Agent runtimes](/concepts/agent-runtimes) before
changing config.

## Quick choice

| Goal                                                 | Use                                                      | Notes                                                                 |
| ---------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------- |
| ChatGPT/Codex subscription with native Codex runtime | `openai/gpt-5.5`                                         | Default OpenAI agent setup. Sign in with Codex auth.                  |
| Direct API-key billing for agent models              | `openai/gpt-5.5` plus a Codex-compatible API-key profile | Use `auth.order.openai` to place the backup after subscription auth.  |
| Direct API-key billing through explicit PI           | `openai/gpt-5.5` plus provider/model runtime `pi`        | Select a normal `openai` API-key profile.                             |
| Latest ChatGPT Instant API alias                     | `openai/chat-latest`                                     | Direct API-key only. Moving alias for experiments, not the default.   |
| ChatGPT/Codex subscription auth through explicit PI  | `openai/gpt-5.5` plus provider/model runtime `pi`        | Select an `openai-codex` auth profile for the compatibility route.    |
| Image generation or editing                          | `openai/gpt-image-2`                                     | Works with either `OPENAI_API_KEY` or OpenAI Codex OAuth.             |
| Transparent-background images                        | `openai/gpt-image-1.5`                                   | Use `outputFormat=png` or `webp` and `openai.background=transparent`. |

## Naming map

The names are similar but not interchangeable:

| Name you see                            | Layer                      | Meaning                                                                                                              |
| --------------------------------------- | -------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `openai`                                | Provider prefix            | Canonical OpenAI model route; agent turns use the Codex runtime.                                                     |
| `openai-codex`                          | Legacy auth/profile prefix | Older OpenAI Codex OAuth/subscription profile namespace. Existing profiles and `auth.order.openai-codex` still work. |
| `codex` plugin                          | Plugin                     | Bundled OpenClaw plugin that provides native Codex app-server runtime and `/codex` chat controls.                    |
| provider/model `agentRuntime.id: codex` | Agent runtime              | Force the native Codex app-server harness for matching embedded turns.                                               |
| `/codex ...`                            | Chat command set           | Bind/control Codex app-server threads from a conversation.                                                           |
| `runtime: "acp", agentId: "codex"`      | ACP session route          | Explicit fallback path that runs Codex through ACP/acpx.                                                             |

This means a config can intentionally contain `openai/*` model refs while auth
profiles still point at Codex-compatible credentials. Prefer `auth.order.openai`
for new config; existing `openai-codex:*` profiles and `auth.order.openai-codex`
remain supported. `openclaw doctor --fix` rewrites legacy `openai-codex/*` model
refs to the canonical OpenAI model route.

> **Note:** GPT-5.5 is available through both direct OpenAI Platform API-key access and
subscription/OAuth routes. For ChatGPT/Codex subscription plus native Codex
execution, use `openai/gpt-5.5`; unset runtime config now selects the Codex
harness for OpenAI agent turns. Use OpenAI API-key profiles only when you want
direct API-key auth for an OpenAI agent model.


> **Note:** OpenAI agent model turns require the bundled Codex app-server plugin. Explicit
PI runtime config remains available as an opt-in compatibility route. When PI is
explicitly selected with an `openai-codex` auth profile, OpenClaw keeps the
public model ref as `openai/*` and routes PI internally through the legacy
Codex-auth transport. Run `openclaw doctor --fix` to repair stale
`openai-codex/*` model refs or old PI session pins that do not come from
explicit runtime config.


## OpenClaw feature coverage

| OpenAI capability         | OpenClaw surface                                                                 | Status                                                 |
| ------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Chat / Responses          | `openai/<model>` model provider                                                  | Yes                                                    |
| Codex subscription models | `openai/<model>` with `openai-codex` OAuth                                       | Yes                                                    |
| Legacy Codex model refs   | `openai-codex/<model>`                                                           | Repaired by doctor to `openai/<model>`                 |
| Codex app-server harness  | `openai/<model>` with omitted runtime or provider/model `agentRuntime.id: codex` | Yes                                                    |
| Server-side web search    | Native OpenAI Responses tool                                                     | Yes, when web search is enabled and no provider pinned |
| Images                    | `image_generate`                                                                 | Yes                                                    |
| Videos                    | `video_generate`                                                                 | Yes                                                    |
| Text-to-speech            | `messages.tts.provider: "openai"` / `tts`                                        | Yes                                                    |
| Batch speech-to-text      | `tools.media.audio` / media understanding                                        | Yes                                                    |
| Streaming speech-to-text  | Voice Call `streaming.provider: "openai"`                                        | Yes                                                    |
| Realtime voice            | Voice Call `realtime.provider: "openai"` / Control UI Talk                       | Yes                                                    |
| Embeddings                | memory embedding provider                                                        | Yes                                                    |

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

**API key (OpenAI Platform):**

**Best for:** direct API access and usage-based billing.

    **Get your API key**

Create or copy an API key from the [OpenAI Platform dashboard](https://platform.openai.com/api-keys).
      
**Run onboarding**

```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        Or pass the key directly:

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      
**Verify the model is available**

```bash
        openclaw models list --provider openai
        ```
      
### Route summary

    | Model ref              | Runtime config             | Route                       | Auth             |
    | ---------------------- | -------------------------- | --------------------------- | ---------------- |
    | `openai/gpt-5.5`      | omitted / provider/model `agentRuntime.id: "codex"` | Codex app-server harness | Codex-compatible OpenAI profile |
    | `openai/gpt-5.4-mini` | omitted / provider/model `agentRuntime.id: "codex"` | Codex app-server harness | Codex-compatible OpenAI profile |
    | `openai/gpt-5.5`      | provider/model `agentRuntime.id: "pi"`              | PI embedded runtime      | `openai` profile or selected `openai-codex` profile |

    > **Note:** `openai/*` agent models use the Codex app-server harness. To use API-key
    auth for an agent model, create a Codex-compatible API-key profile and order
    it with `auth.order.openai`; `OPENAI_API_KEY` remains the direct fallback for
    non-agent OpenAI API surfaces. Older `auth.order.openai-codex` entries still
    work.
    

    ### Config example

    ```json5
    {
      env: { OPENAI_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "openai/gpt-5.5" } } },
    }
    ```

    To try ChatGPT's current Instant model from the OpenAI API, set the model
    to `openai/chat-latest`:

    ```json5
    {
      env: { OPENAI_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` is a moving alias. OpenAI documents it as the latest Instant
    model used in ChatGPT and recommends `gpt-5.5` for production API usage, so
    keep `openai/gpt-5.5` as the stable default unless you explicitly want that
    alias behavior. The alias currently accepts only `medium` text verbosity, so
    OpenClaw normalizes incompatible OpenAI text-verbosity overrides for this
    model.

    > **Note:** OpenClaw does **not** expose `openai/gpt-5.3-codex-spark`. Live OpenAI API requests reject that model, and the current Codex catalog does not expose it either.
    

  
**Codex subscription:**

**Best for:** using your ChatGPT/Codex subscription with native Codex app-server execution instead of a separate API key. Codex cloud requires ChatGPT sign-in.

    **Run Codex OAuth**

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
      
**Use the canonical OpenAI model route**

```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.5
        ```

        No runtime config is required for the default path. OpenAI agent turns
        select the native Codex app-server runtime automatically, and OpenClaw
        installs or repairs the bundled Codex plugin when this route is chosen.
      
**Verify Codex auth is available**

```bash
        openclaw models list --provider openai-codex
        ```

        After the gateway is running, send `/codex status` or `/codex models`
        in chat to verify the native app-server runtime.
      
### Route summary

    | Model ref | Runtime config | Route | Auth |
    |-----------|----------------|-------|------|
    | `openai/gpt-5.5` | omitted / provider/model `agentRuntime.id: "codex"` | Native Codex app-server harness | Codex sign-in or ordered `openai` auth profile |
    | `openai/gpt-5.5` | provider/model `agentRuntime.id: "pi"` | PI embedded runtime with internal Codex-auth transport | Selected `openai-codex` profile |
    | `openai-codex/gpt-5.5` | repaired by doctor | Legacy route rewritten to `openai/gpt-5.5` | Existing `openai-codex` profile |

    > **Note:** Do not configure older `openai-codex/gpt-5.1*`, `openai-codex/gpt-5.2*`, or
    `openai-codex/gpt-5.3*` model refs. ChatGPT/Codex OAuth accounts now reject
    those models. Use `openai/gpt-5.5`; OpenAI agent turns now select the Codex
    runtime by default.
    

    > **Note:** The `openai-codex/*` model prefix is legacy config repaired by doctor. For
    the common subscription plus native runtime setup, sign in with Codex auth
    but keep the model re
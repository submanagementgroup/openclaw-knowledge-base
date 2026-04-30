---
domain: concepts
topic: "Model Configuration: Provider/Model Format, Default Model, and Model Selection"
type: reference
keywords:
  - models
  - model selection
  - model identifier
  - agents.defaults.model
  - openclaw models list
  - provider/model
  - model config
related:
  - concepts/model-failover
  - providers/openai
  - providers/anthropic
  - providers/ollama
source:
  - concepts/models.md
  - concepts/model-providers.md
---

Model configuration in OpenClaw: how to select a model, set per-agent models, and use custom providers. Models are identified by `provider/model-name` strings.

## Model Identifier Format

Models are specified as `provider/model-name`:
- `openai/gpt-4o`
- `anthropic/claude-opus-4-5`
- `ollama/llama3.2`
- `google/gemini-2.5-pro`

## Setting the Default Model

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-4o",
    }
  }
}
```

## Model Selection and Listing

```bash
openclaw models list              # list all available models
openclaw models list --provider openai
openclaw infer "test" --model anthropic/claude-opus-4-5
```

Auth profile rotation, cooldowns, and how that interacts with fallbacks.

    Quick provider overview and examples.

    PI, Codex, and other agent loop runtimes.

    Model config keys.

Model refs choose a provider and model. They do not usually choose the low-level agent runtime. For example, `openai/gpt-5.5` can run through the normal OpenAI provider path or through the Codex app-server runtime, depending on `agents.defaults.agentRuntime.id`. See [Agent runtimes](/concepts/agent-runtimes).

## How model selection works

OpenClaw selects models in this order:

    `agents.defaults.model.primary` (or `agents.defaults.model`).

    `agents.defaults.model.fallbacks` (in order).

    Auth failover happens inside a provider before moving to the next model.

    - `agents.defaults.models` is the allowlist/catalog of models OpenClaw can use (plus aliases).
    - `agents.defaults.imageModel` is used **only when** the primary model can't accept images.
    - `agents.defaults.pdfModel` is used by the `pdf` tool. If omitted, the tool falls back to `agents.defaults.imageModel`, then the resolved session/default model.
    - `agents.defaults.imageGenerationModel` is used by the shared image-generation capability. If omitted, `image_generate` can still infer an auth-backed provider default. It tries the current default provider first, then the remaining registered image-generation providers in provider-id order. If you set a specific provider/model, also configure that provider's auth/API key.
    - `agents.defaults.musicGenerationModel` is used by the shared music-generation capability. If omitted, `music_generate` can still infer an auth-backed provider default. It tries the current default provider first, then the remaining registered music-generation providers in provider-id order. If you set a specific provider/model, also configure that provider's auth/API key.
    - `agents.defaults.videoGenerationModel` is used by the shared video-generation capability. If omitted, `video_generate` can still infer an auth-backed provider default. It tries the current default provider first, then the remaining registered video-generation providers in provider-id order. If you set a specific provider/model, also configure that provider's auth/API key.
    - Per-agent defaults can override `agents.defaults.model` via `agents.list[].model` plus bindings (see [Multi-agent routing](/concepts/multi-agent)).

## Selection source and fallback behavior

The same `provider/model` can mean different things depending on where it came from:

- Configured defaults (`agents.defaults.model.primary` and agent-specific primaries) are the normal starting point and use `agents.defaults.model.fallbacks`.
- Auto fallback selections are temporary recovery state. They are stored with `modelOverrideSource: "auto"` so later turns can keep using the fallback chain without probing a known-bad primary first.
- User session selections are exact. `/model`, the model picker, `session_status(model=...)`, and `sessions.patch` store `modelOverrideSource: "user"`; if that selected provider/model is unreachable, OpenClaw fails visibly instead of falling through to another configured model.
- Cron `--model` / payload `model` is a per-job primary. It still uses configured fallbacks unless the job supplies explicit payload `fallbacks` (use `fallbacks: []` for a strict cron run).
- CLI default-model and allowlist pickers respect `models.mode: "replace"` by listing explicit `models.providers.*.models` instead of loading the full built-in catalog.
- The Control UI model picker asks the Gateway for its configured model view: `agents.defaults.models` when present, otherwise explicit `models.providers.*.models` plus providers with usable auth. The full built-in catalog is reserved for explicit browse views such as `models.list` with `view: "all"` or `openclaw models list --all`.

## Quick model policy

- Set your primary to the strongest latest-generation model available to you.
- Use fallbac

## Built-in Providers Summary

Reference for **LLM/model providers** (not chat channels like WhatsApp/Telegram). For model selection rules, see [Models](/concepts/models).

## Quick rules

    - Model refs use `provider/model` (example: `opencode/claude-opus-4-6`).
    - `agents.defaults.models` acts as an allowlist when set.
    - CLI helpers: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
    - `models.providers.*.contextWindow` / `contextTokens` / `maxTokens` set provider-level defaults; `models.providers.*.models[].contextWindow` / `contextTokens` / `maxTokens` override them per model.
    - Fallback rules, cooldown probes, and session-override persistence: [Model failover](/concepts/model-failover).

    OpenAI-family routes are prefix-specific:

    - `openai/<model>` uses the direct OpenAI API-key provider in PI.
    - `openai-codex/<model>` uses Codex OAuth in PI.
    - `openai/<model>` plus `agents.defaults.agentRuntime.id: "codex"` uses the native Codex app-server harness.

    See [OpenAI](/providers/openai) and [Codex harness](/plugins/codex-harness). If the provider/runtime split is confusing, read [Agent runtimes](/concepts/agent-runtimes) first.

    Plugin auto-enable follows the same boundary: `openai-codex/<model>` belongs to the OpenAI plugin, while the Codex plugin is enabled by `agentRuntime.id: "codex"` or legacy `codex/<model>` refs.

    GPT-5.5 is available through `openai/gpt-5.5` for direct API-key traffic, `openai-codex/gpt-5.5` in PI for Codex OAuth, and the native Codex app-server harness when `agentRuntime.id: "codex"` is set.

    CLI runtimes use the same split: choose canonical model refs such as `anthropic/claude-*`, `google/gemini-*`, or `openai/gpt-*`, then set `agents.defaults.agentRuntime.id` to `claude-cli`, `google-gemini-cli`, or `codex-cli` when you want a local CLI backend.

    Legacy `claude-cli/*`, `google-gemini-cli/*`, and `codex-cli/*` refs migrate back to canonical provider refs with the runtime recorded separately.

## Plugin-owned provider behavior

Most provider-specific logic lives in provider plugins (`registerProvider(...)`) while OpenClaw keeps the generic inference loop. Plugins own onboarding, model catalogs, auth env-var mapping, transport/config normalization, tool-schema cleanup, failover classification, OAuth refresh, usage reporting, thinking/reasoning profiles, and more.

The full list of provider-SDK hooks and bundled-plugin examples lives in [Provider plugins](/plugins/sdk-provider-plugins). A provider that needs a totally custom request executor is a separate, deeper extension surface.

Provider-owned runner behavior lives on explicit provider hooks such as replay policy, tool-schema normalization, stream wrapping, and transport/request helpers. The legacy `ProviderPlugin.capabilities` static bag is compatibility-only and is no longer read by shared runner logic.

## API key rotation

    Configure multiple keys via:

    - `OPENCLAW_LIVE_<PROVIDER>_KEY` (single live overr

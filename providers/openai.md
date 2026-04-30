---
domain: providers
topic: "OpenAI Provider: GPT-4o, Image Generation, Voice, Codex, and Azure OpenAI Setup"
type: procedure
keywords:
  - OpenAI
  - gpt-4o
  - OPENAI_API_KEY
  - OpenAI provider
  - image generation
  - Azure OpenAI
  - Codex
related:
  - providers/anthropic
  - providers/google
  - concepts/models
source: providers/openai.md
---

OpenAI is a primary AI provider in OpenClaw supporting chat, image generation, video, voice, and embeddings. Configure with `OPENAI_API_KEY` or an OAuth session.

## Quick Setup

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      model: "openai/gpt-4o",
    }
  }
}
```

Set `OPENAI_API_KEY` environment variable, or use `openclaw configure` to enter it interactively.

## Available Models

Key models: `gpt-4o`, `gpt-4o-mini`, `gpt-4.5`, `gpt-4.1`, `o3`, `o4-mini`, `codex-mini-latest`

## Feature Coverage

OpenAI provides developer APIs for GPT models, and Codex is also available as a
ChatGPT-plan coding agent through OpenAI's Codex clients. OpenClaw keeps those
surfaces separate so config stays predictable.

OpenClaw supports three OpenAI-family routes. The model prefix selects the
provider/auth route; a separate runtime setting selects who executes the
embedded agent loop:

- **API key** — direct OpenAI Platform access with usage-based billing (`openai/*` models)
- **Codex subscription through PI** — ChatGPT/Codex sign-in with subscription access (`openai-codex/*` models)
- **Codex app-server harness** — native Codex app-server execution (`openai/*` models plus `agents.defaults.agentRuntime.id: "codex"`)

OpenAI explicitly supports subscription OAuth usage in external tools and workflows like OpenClaw.

Provider, model, runtime, and channel are separate layers. If those labels are
getting mixed together, read [Agent runtimes](/concepts/agent-runtimes) before
changing config.

## Quick choice

| Goal                                          | Use                                              | Notes                                                                        |
| --------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------- |
| Direct API-key billing                        | `openai/gpt-5.5`                                 | Set `OPENAI_API_KEY` or run OpenAI API-key onboarding.                       |
| GPT-5.5 with ChatGPT/Codex subscription auth  | `openai-codex/gpt-5.5`                           | Default PI route for Codex OAuth. Best first choice for subscription setups. |
| GPT-5.5 with native Codex app-server behavior | `openai/gpt-5.5` plus `agentRuntime.id: "codex"` | Forces the Codex app-server harness for that model ref.                      |
| Image generation or editing                   | `openai/gpt-image-2`                             | Works with either `OPENAI_API_KEY` or OpenAI Codex OAuth.                    |
| Transparent-background images                 | `openai/gpt-image-1.5`                           | Use `outputFormat=png` or `webp` and `openai.background=transparent`.        |

## Naming map

The names are similar but not interchangeable:

| Name you see                       | Layer             | Meaning                                                                                           |
| ---------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------- |
| `openai`                           | Provider prefix   | Direct OpenAI Platform API route.                                                                 |
| `openai-codex`                     | Provider prefix   | OpenAI Codex OAuth/subscription route through the normal OpenClaw PI runner.                      |
| `codex` plugin                     | Plugin            | Bundled OpenClaw plugin that provides native Codex app-server runtime and `/codex` chat controls. |
| `agentRuntime.id: codex`           | Agent runtime     | Force the native Codex app-server harness for embedded turns.                                     |
| `/codex ...`                       | Chat command set  | Bind/control Codex app-server threads from a conversation.                                        |
| `runtime: "acp", agentId: "codex"` | ACP session route | Explicit fallback path that runs Codex through ACP/acpx.                                          |

This means a config can intentionally contain both `openai-codex/*` and the
`codex` plugin. That is valid when you want Codex OAuth through PI and also want
native `/codex` chat controls available. `openclaw doctor` warns about that
combination so you can confirm it is intentional; it does not rewrite it.

GPT-5.5 is available through both direct OpenAI Platform API-key access and
subscription/OAuth routes. Use `openai/gpt-5.5` for direct `OPENAI_API_KEY`
traffic, `openai-codex/gpt-5.5` for Codex OAuth through PI, or
`openai/gpt-5.5` with `agentRuntime.id: "codex"` for the native Codex
app-server harness.

Enabling the OpenAI plugin, or selecting an `openai-codex/*` model, does not
enable the bundled Codex app-server plugin. OpenClaw enables that plugin only
when you explicitly select the native Codex harness with
`agentRuntime.id: "codex"` or use a legacy `codex/*` model ref.
If the bundled `codex` plugin is enabled but `openai-codex/*` still resolves
through PI, `openclaw doctor` warns and leaves the route unchanged.

## OpenClaw feature coverage

| OpenAI capability         | OpenClaw surface                                           | Status                                                 |
| ------------------------- | ---------------------------------------------------------- | ------------------------------------------------------ |
| Chat / Responses          | `op

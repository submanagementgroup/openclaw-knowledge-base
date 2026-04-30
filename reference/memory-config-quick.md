---
domain: reference
topic: "Memory Configuration Quick Reference: All memorySearch and QMD Fields"
type: reference
keywords:
  - memory config reference
  - memorySearch fields
  - QMD config fields
  - memory configuration
related:
  - memory/memory-search
  - memory/memory-config-reference
source: reference/memory-config.md
---

Quick reference for all memory configuration fields in OpenClaw. For full documentation see memory/ domain files.

This page lists every configuration knob for OpenClaw memory search. For conceptual overviews, see:

    How memory works.

    Default SQLite backend.

    Local-first sidecar.

    Search pipeline and tuning.

    Memory sub-agent for interactive sessions.

All memory search settings live under `agents.defaults.memorySearch` in `openclaw.json` unless noted otherwise.

If you are looking for the **active memory** feature toggle and sub-agent config, that lives under `plugins.entries.active-memory` instead of `memorySearch`.

Active memory uses a two-gate model:

1. the plugin must be enabled and target the current agent id
2. the request must be an eligible interactive persistent chat session

See [Active Memory](/concepts/active-memory) for the activation model, plugin-owned config, transcript persistence, and safe rollout pattern.

---

## Provider selection

| Key        | Type      | Default          | Description                                                                                                                                                                                                                        |
| ---------- | --------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider` | `string`  | auto-detected    | Embedding adapter ID such as `bedrock`, `deepinfra`, `gemini`, `github-copilot`, `local`, `mistral`, `ollama`, `openai`, or `voyage`; may also be a configured `models.providers.<id>` whose `api` points at one of those adapters |
| `model`    | `string`  | provider default | Embedding model name                                                                                                                                                                                                               |
| `fallback` | `string`  | `"none"`         | Fallback adapter ID when the primary fails                                                                                                                                                                                         |
| `enabled`  | `boolean` | `true`           | Enable or disable memory search                                                                                                                                                                                                    |

### Auto-detection order

When `provider` is not set, OpenClaw selects the first available:

    Selected if `memorySearch.local.modelPath` is configured and the file exists.

    Selected if a GitHub Copilot token can be resolved (env var or auth profile).

    Selected if an OpenAI key can be resolved.

    Selected if a Gemini key can be resolved.

    Selected if a Voyage key can be resolved.

    Selected if a Mistral key can be resolved.

    Selected if a DeepInfra key can be resolved.

    Selected if the AWS SDK credential chain resolves (instance role, access keys, profile, SSO, web identity, or shared config).

`ollama` is supported but not auto-detected (set it explicitly).

### Custom provider ids

`memorySearch.provider` can point at a custom `models.providers.<id>` entry. OpenClaw resolves that provider's `api` owner for the embedding adapter while preserving the custom provider id for endpoint, auth, and model-prefix handling. This lets multi-GPU or multi-host setups dedicate memory embeddings to a specific local endpoint:

```json5
{
  models: {
    providers: {
      "ollama-5080": {
        api: "ollama",
        baseUrl: "http://gpu-box.local:11435",
        apiKey: "ollama-local",
        models: [{ id: "qwen3-embedding:0.6b" }],
      },
    },
  },
  agents: {
    defaults: {
      memorySearch: {
        provider: "ollama-5080",
        model: "qwen3-embedding:0.6b",
      },
    },
  },
}
```

### API key resolution

Remote embeddings require an API key. Bedrock uses the AWS SDK default credential chain instead (instance roles, SSO, access keys).

| Provider       | Env var                                            | Config key                          |
| -------------- | -------------------------------------------------- | ----------------------------------- |
| Bedrock        | AWS credential chain                               | No API key needed                   |
| DeepInfra      | `DEEPINFRA_API_KEY`                                | `models.providers.deepinfra.apiKey` |
| Gemini         | `GEMINI_API_KEY`                                   | `models.providers.google.apiKey`    |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN` | Auth profile via device login       |
| Mistral        | `MISTRAL_API_KEY`                                  | `models.providers.mistral.apiKey`   |
| Ollama         | `OLLAMA_API_KEY` (placeholder)                     | --

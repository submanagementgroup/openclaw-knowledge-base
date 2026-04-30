---
domain: memory
topic: "Memory Configuration Reference: memorySearch, QMD, Active Memory, and Embedding Settings"
type: reference
keywords:
  - memory config
  - memorySearch config
  - QMD config
  - embedding provider config
  - memory reference
  - memory settings
related:
  - memory/memory-search
  - memory/memory-qmd
  - memory/active-memory
source: reference/memory-config.md
---

Complete configuration reference for all memory-related settings in OpenClaw. Covers memorySearch, QMD, active memory, and memory file paths.

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
| Ollama         | `OLLAMA_API_KEY` (placeholder)                     | --                                  |
| OpenAI         | `OPENAI_API_KEY`                                   | `models.providers.openai.apiKey`    |
| Voyage         | `VOYAGE_API_KEY`                                   | `models.providers.voyage.apiKey`    |

Codex OAuth covers chat/completions only and does not satisfy embedding requests.

---

## Remote endpoint config

For custom OpenAI-compatible endpoints or overriding provider defaults:

<ParamField path="remote.baseUrl" type="string">
  Custom API base URL.
</ParamField>
<ParamField path="remote.apiKey" type="string">
  Override API key.
</ParamField>
<ParamField path="remote.headers" type="object">
  Extra HTTP headers (merged with provider defaults).
</ParamField>

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Provider-specific config

    | Key                    | Type     | Default                | Description                                |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------ |
    | `model`                | `string` | `gemini-embedding-001` | Also supports `gemini-embedding-2-preview` |
    | `outputDimensionality` | `number` | `3072`                 | For Embedding 2: 768, 1536, or 3072        |

    Changing model or `outputDimensionality` triggers an automatic full reindex.

    OpenAI-compatible embedding endpoints can opt into provider-specific `input_type` request fields. This is useful for asymmetric embedding models that require different labels for query and document embeddings.

    | Key                 | Type     | Default | Description                                             |
    | ------------------- | -------- | ------- | ------------------------------------------------------- |
    | `inputType`         | `string` | unset   | Shared `input_type` for query and document embeddings   |
    | `queryInputType`    | `string` | unset   | Query-time `input_type`; overrides `inputType`          |
    | `documentInputType` | `string` | unset   | Index/document `input_type`; overrides `inputType`      |

    ```json5
    {
      agents: {
        defaults: {
          memorySearch: {
            provider: "openai",
            remote: {
              baseUrl: "https://embeddings.example/v1",
              apiKey: "env:EMBEDDINGS_API_KEY",
            },
            model: "asymmetric-embedder",
            queryInputType: "query",
            documentInputType: "passage",
          },
        },
      },
    }
    ```

    Changing these values affects embedding cache identity for provider batch indexing and should be followed by a memory reindex when the upstream model treats the labels differently.

    Bedrock uses the AWS SDK default credential chain — no API keys needed. If OpenClaw runs on EC2 with a Bedrock-enabled instance role, just set the provider and model:

    ```json5
    {
      agents: {
        defaults: {
          memorySearch: {
            provider: "bedrock",
            model: "amazon.titan-embed-text-v2:0",
          },
        },
      },
    }
    ```

    | Key                    | Type     | Default                        | Description                     |
    | ---------------------- | -------- | ------------------------------ | ------------------------------- |
    | `model`                | `string` | `amazon.titan-embed-text-v2:0` | Any Bedrock embedding model ID  |
    | `outputDimensionality` | `number` | model default                  | For Titan V2: 256, 512, or 1024 |

    **Supported models** (with family detection and dimension defaults):

    | Model ID                                   | Provider   | Default Dims | Configurable Dims    |
    | ------------------------------------------ | ---------- | ------------ | -------------------- |
    | `a

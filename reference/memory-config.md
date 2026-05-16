---
domain: reference
topic: "Memory Configuration Reference"
type: reference
keywords:
  - memory config
  - memorySearch config
  - memory reference
  - memory backend config
  - embedding config
source: reference/memory-config.md
---

This page lists every configuration knob for OpenClaw memory search. For conceptual overviews, see:

**Memory overview:** How memory works.
  

**Builtin engine:** Default SQLite backend.
  

**QMD engine:** Local-first sidecar.
  

**Memory search:** Search pipeline and tuning.
  

**Active memory:** Memory sub-agent for interactive sessions.
  

All memory search settings live under `agents.defaults.memorySearch` in `openclaw.json` unless noted otherwise.

> **Note:** If you are looking for the **active memory** feature toggle and sub-agent config, that lives under `plugins.entries.active-memory` instead of `memorySearch`.

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

**local**

Selected if `memorySearch.local.modelPath` is configured and the file exists.
  
**github-copilot**

Selected if a GitHub Copilot token can be resolved (env var or auth profile).
  
**openai**

Selected if an OpenAI key can be resolved.
  
**gemini**

Selected if a Gemini key can be resolved.
  
**voyage**

Selected if a Voyage key can be resolved.
  
**mistral**

Selected if a Mistral key can be resolved.
  
**deepinfra**

Selected if a DeepInfra key can be resolved.
  
**bedrock**

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

> **Note:** Codex OAuth covers chat/completions only and does not satisfy embedding requests.


---

## Remote endpoint config

For custom OpenAI-compatible endpoints or overriding provider defaults:


  Custom API base URL.


  Override API key.


  Extra HTTP headers (merged with provider defaults).


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

### Gemini

| Key                    | Type     | Default                | Description                                |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------ |
    | `model`                | `string` | `gemini-embedding-001` | Also supports `gemini-embedding-2-preview` |
    | `outputDimensionality` | `number` | `3072`                 | For Embedding 2: 768, 1536, or 3072        |

    > **Note:** Changing model or `outputDimensionality` triggers an automatic full reindex.
    

  ### OpenAI-compatible input types

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

  ### Bedrock

### Bedrock embedding config

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
    | `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024       |
    | `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                   |
    | `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                   |
    | `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                   |
    | `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072 |
    | `cohere.embed-english-v3`                  | Cohere     | 1024         | --                   |
    | `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                   |
    | `cohere.embed-v4:0`                        | Cohere     | 1536         | 256-1536             |
    | `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                   |
    | `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                   |

    Throughput-suffixed variants (e.g., `amazon.titan-embed-text-v1:2:8k`) inherit the base model's configuration.

    **Authentication:** Bedrock auth uses the standard AWS SDK credential resolution order:

    1. Environment variables (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
    2. SSO token cache
    3. Web identity token credentials
    4. Shared credentials and config files
    5. ECS or EC2 metadata credentials

    Region is resolved from `AWS_REGION`, `AWS_DEFAULT_REGION`, the `amazon-bedrock` provider `baseUrl`, or defaults to `us-east-1`.

    **IAM permissions:** the IAM role or user needs:

    ```json
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "*"
    }
    ```

    For least-privilege, scope `InvokeModel` to the specific model:

    ```
    arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
    ```

  ### Local (GGUF + node-llama-cpp)

| Key                   | Type               | Default                | Description                                                                                                                                                                                                                                                                                                          |
    | --------------------- | ------------------ | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `local.modelPath`     | `string`           | auto-downloaded        | Path to GGUF model file                                                            
---
domain: providers
topic: "NVIDIA NIM Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - NVIDIA NIM
  - nvidia provider
  - AI provider
  - model config
  - API key
  - NVIDIA NIM
  - NIM
  - NVIDIA inference
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/nvidia.md
---

NVIDIA NIM provider setup, configuration, and model reference for OpenClaw.

NVIDIA provides an OpenAI-compatible API at `https://integrate.api.nvidia.com/v1` for
open models for free. Authenticate with an API key from
[build.nvidia.com](https://build.nvidia.com/settings/api-keys).

## Getting started

    Create an API key at [build.nvidia.com](https://build.nvidia.com/settings/api-keys).

    ```bash
    export NVIDIA_API_KEY="nvapi-..."
    openclaw onboard --auth-choice nvidia-api-key
    ```

    ```bash
    openclaw models set nvidia/nvidia/nemotron-3-super-120b-a12b
    ```

If you pass `--nvidia-api-key` instead of the env var, the value lands in shell
history and `ps` output. Prefer the `NVIDIA_API_KEY` environment variable when
possible.

For non-interactive setup, you can also pass the key directly:

```bash
openclaw onboard --auth-choice nvidia-api-key --nvidia-api-key "nvapi-..."
```

## Config example

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-super-120b-a12b" },
    },
  },
}
```

## Built-in catalog

| Model ref                                  | Name                         | Context | Max output |
| ------------------------------------------ | ---------------------------- | ------- | ---------- |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | NVIDIA Nemotron 3 Super 120B | 262,144 | 8,192      |
| `nvidia/moonshotai/kimi-k2.5`              | Kimi K2.5                    | 262,144 | 8,192      |
| `nvidia/minimaxai/minimax-m2.5`            | Minimax M2.5                 | 196,608 | 8,192      |
| `nvidia/z-ai/glm5`                         | GLM 5                        | 202,752 | 8,192      |

## Advanced configuration

    The provider auto-enables when the `NVIDIA_API_KEY` environment variable is set.
    No explicit provider config is required beyond the key.

    The bundled catalog is static. Costs default to `0` in source since NVIDIA
    currently offers free API access for the listed models.

    NVIDIA uses the standard `/v1` completions endpoint. Any OpenAI-compatible
    tooling should work out of the box with the NVIDIA base URL.

NVIDIA models are currently free to use. Check
[build.nvidia.com](https://build.nvidia.com/) for the latest availability and
rate-limit details.

## Related

    Choosing providers, model refs, and failover behavior.

    Full config reference for agents, models, and providers.

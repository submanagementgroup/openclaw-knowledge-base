---
domain: providers
topic: "Ollama Provider: Local LLM Setup, Auto-Discovery, Vision, and Web Search"
type: procedure
keywords:
  - Ollama
  - local models
  - ollama provider
  - llama3
  - local LLM
  - Ollama auto-discovery
  - ollama config
related:
  - providers/lmstudio
  - providers/vllm
  - gateway/local-models
source: providers/ollama.md
---

Ollama runs large language models locally on your machine. OpenClaw auto-discovers a running Ollama instance and lists its models automatically.

## Quick Setup

```bash
# Install Ollama
brew install ollama        # macOS
curl -fsSL https://ollama.ai/install.sh | sh  # Linux

# Pull a model
ollama pull llama3.2

# Ollama auto-discovered, or configure explicitly:
```

```json5
{
  agents: {
    defaults: {
      model: "ollama/llama3.2",
    }
  }
}
```

## Auto-Discovery

OpenClaw auto-discovers Ollama at `http://localhost:11434`. Run `openclaw models list` to see available models.

## Configuration and Tuning

OpenClaw integrates with Ollama's native API (`/api/chat`) for hosted cloud models and local/self-hosted Ollama servers. You can use Ollama in three modes: `Cloud + Local` through a reachable Ollama host, `Cloud only` against `https://ollama.com`, or `Local only` against a reachable Ollama host.

**Remote Ollama users**: Do not use the `/v1` OpenAI-compatible URL (`http://host:11434/v1`) with OpenClaw. This breaks tool calling and models may output raw tool JSON as plain text. Use the native Ollama API URL instead: `baseUrl: "http://host:11434"` (no `/v1`).

Ollama provider config uses `baseUrl` as the canonical key. OpenClaw also accepts `baseURL` for compatibility with OpenAI SDK-style examples, but new config should prefer `baseUrl`.

## Auth rules

    Local and LAN Ollama hosts do not need a real bearer token. OpenClaw uses the local `ollama-local` marker only for loopback, private-network, `.local`, and bare-hostname Ollama base URLs.

    Remote public hosts and Ollama Cloud (`https://ollama.com`) require a real credential through `OLLAMA_API_KEY`, an auth profile, or the provider's `apiKey`.

    Custom provider ids that set `api: "ollama"` follow the same rules. For example, an `ollama-remote` provider that points at a private LAN Ollama host can use `apiKey: "ollama-local"` and sub-agents will resolve that marker through the Ollama provider hook instead of treating it as a missing credential. Memory search can also set `agents.defaults.memorySearch.provider` to that custom provider id so embeddings use the matching Ollama endpoint.

    `auth-profiles.json` stores the credential for a provider id. Put endpoint settings (`baseUrl`, `api`, model ids, headers, timeouts) in `models.providers.<id>`. Older flat auth-profile files such as `{ "ollama-windows": { "apiKey": "ollama-local" } }` are not a runtime format; run `openclaw doctor --fix` to rewrite them to the canonical `ollama-windows:default` API-key profile with a backup. `baseUrl` in that file is compatibility noise and should be moved to provider config.

    When Ollama is used for memory embeddings, bearer auth is scoped to the host where it was declared:

    - A provider-level key is sent only to that provider's Ollama host.
    - `agents.*.memorySearch.remote.apiKey` is sent only to its remote embedding host.
    - A pure `OLLAMA_API_KEY` env value is treated as the Ollama Cloud convention, not sent to local or self-hosted hosts by default.

## Getting started

Choose your preferred setup method and mode.

    **Best for:** fastest path to a working Ollama cloud or local setup.

        ```bash
        openclaw onboard
        ```

        Select **Ollama** from the provider list.

        - **Cloud + Local** — local Ollama host plus cloud models routed through that host
        - **Cloud only** — hosted Ollama models via `https://ollama.com`
        - **Local only** — local models only

        `Cloud only` prompts for `OLLAMA_API_KEY` and suggests hosted cloud defaults. `Cloud + Local` and `Local only` ask for an Ollama base URL, discover available models, and auto-pull the selected local model if it is not available yet. When Ollama reports an installed `:latest` tag such as `gemma4:latest`, setup shows that installed model once instead of showing both `gemma4` and `gemma4:latest` or pulling the bare alias again. `Cloud + Local` also checks whether that Ollama host is signed in for cloud access.

        ```bash
        openclaw models list --provider ollama
        ```

    ### Non-interactive mode

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --accept-risk
    ```

    Optionally specify a custom base URL or model:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    **Best for:** full control over cloud or local setup.

        - **Cloud + Local**: install Ollama, sign in with `ollama signin`, and route cloud requests through that host
        - **Cloud only**: use `https://ollama.com` with an `OLLAMA_API_KEY`
        - **Local only**: install Ollama from [ollama.com/download](https://ollama.com/download)

        ```bash
        ollama pull gemma4
        # or
        ollama pull gpt-oss:20b
        # or
        ollama pull llama3.3
        ```

        For `Cloud only`, use your real `OLLAMA_API_KEY`. For host-backed setups, any placeholder value works:

        ```bash
        # Cloud
        export OLLAMA_API_KEY="your-ollama-api-key"

        # Local-only
        export OLLAMA_API_KEY="ollama-local"

        # Or configure in your config file
        openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"
        ```

        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Or set the default in config:

        ```json5
        {
          agents: {
            defaults

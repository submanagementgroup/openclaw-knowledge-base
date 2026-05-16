---
domain: providers
topic: "Ollama Local Model Provider"
type: integration
keywords:
  - ollama
  - local models
  - ollama pull
  - ollama serve
  - OLLAMA_API_URL
  - local inference
  - embedding ollama
source: providers/ollama.md
---

OpenClaw integrates with Ollama's native API (`/api/chat`) for hosted cloud models and local/self-hosted Ollama servers. You can use Ollama in three modes: `Cloud + Local` through a reachable Ollama host, `Cloud only` against `https://ollama.com`, or `Local only` against a reachable Ollama host.

> **Note:** **Remote Ollama users**: Do not use the `/v1` OpenAI-compatible URL (`http://host:11434/v1`) with OpenClaw. This breaks tool calling and models may output raw tool JSON as plain text. Use the native Ollama API URL instead: `baseUrl: "http://host:11434"` (no `/v1`).


Ollama provider config uses `baseUrl` as the canonical key. OpenClaw also accepts `baseURL` for compatibility with OpenAI SDK-style examples, but new config should prefer `baseUrl`.

## Auth rules

### Local and LAN hosts

Local and LAN Ollama hosts do not need a real bearer token. OpenClaw uses the local `ollama-local` marker only for loopback, private-network, `.local`, and bare-hostname Ollama base URLs.
  ### Remote and Ollama Cloud hosts

Remote public hosts and Ollama Cloud (`https://ollama.com`) require a real credential through `OLLAMA_API_KEY`, an auth profile, or the provider's `apiKey`.
  ### Custom provider ids

Custom provider ids that set `api: "ollama"` follow the same rules. For example, an `ollama-remote` provider that points at a private LAN Ollama host can use `apiKey: "ollama-local"` and sub-agents will resolve that marker through the Ollama provider hook instead of treating it as a missing credential. Memory search can also set `agents.defaults.memorySearch.provider` to that custom provider id so embeddings use the matching Ollama endpoint.
  ### Auth profiles

`auth-profiles.json` stores the credential for a provider id. Put endpoint settings (`baseUrl`, `api`, model ids, headers, timeouts) in `models.providers.<id>`. Older flat auth-profile files such as `{ "ollama-windows": { "apiKey": "ollama-local" } }` are not a runtime format; run `openclaw doctor --fix` to rewrite them to the canonical `ollama-windows:default` API-key profile with a backup. `baseUrl` in that file is compatibility noise and should be moved to provider config.
  ### Memory embedding scope

When Ollama is used for memory embeddings, bearer auth is scoped to the host where it was declared:

    - A provider-level key is sent only to that provider's Ollama host.
    - `agents.*.memorySearch.remote.apiKey` is sent only to its remote embedding host.
    - A pure `OLLAMA_API_KEY` env value is treated as the Ollama Cloud convention, not sent to local or self-hosted hosts by default.

  ## Getting started

Choose your preferred setup method and mode.

**Onboarding (recommended):**

**Best for:** fastest path to a working Ollama cloud or local setup.

    **Run onboarding**

```bash
        openclaw onboard
        ```

        Select **Ollama** from the provider list.
      
**Choose your mode**

- **Cloud + Local** — local Ollama host plus cloud models routed through that host
        - **Cloud only** — hosted Ollama models via `https://ollama.com`
        - **Local only** — local models only

      
**Select a model**

`Cloud only` prompts for `OLLAMA_API_KEY` and suggests hosted cloud defaults. `Cloud + Local` and `Local only` ask for an Ollama base URL, discover available models, and auto-pull the selected local model if it is not available yet. When Ollama reports an installed `:latest` tag such as `gemma4:latest`, setup shows that installed model once instead of showing both `gemma4` and `gemma4:latest` or pulling the bare alias again. `Cloud + Local` also checks whether that Ollama host is signed in for cloud access.
      
**Verify the model is available**

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

  
**Manual setup:**

**Best for:** full control over cloud or local setup.

    **Choose cloud or local**

- **Cloud + Local**: install Ollama, sign in with `ollama signin`, and route cloud requests through that host
        - **Cloud only**: use `https://ollama.com` with an `OLLAMA_API_KEY`
        - **Local only**: install Ollama from [ollama.com/download](https://ollama.com/download)

      
**Pull a local model (local only)**

```bash
        ollama pull gemma4
        # or
        ollama pull gpt-oss:20b
        # or
        ollama pull llama3.3
        ```
      
**Enable Ollama for OpenClaw**

For `Cloud only`, use your real `OLLAMA_API_KEY`. For host-backed setups, any placeholder value works:

        ```bash
        # Cloud
        export OLLAMA_API_KEY="your-ollama-api-key"

        # Local-only
        export OLLAMA_API_KEY="ollama-local"

        # Or configure in your config file
        openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"
        ```
      
**Inspect and set your model**

```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Or set the default in config:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      
## Cloud models

**Cloud + Local:**

`Cloud + Local` uses a reachable Ollama host as the control point for both local and cloud models. This is Ollama's preferred hybrid flow.

    Use **Cloud + Local** during setup. OpenClaw prompts for the Ollama base URL, discovers local models from that host, and checks whether the host is signed in for cloud access with `ollama signin`. When the host is signed in, OpenClaw also suggests hosted cloud defaults such as `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, and `glm-5.1:cloud`.

    If the host is not signed in yet, OpenClaw keeps the setup local-only until you run `ollama signin`.

  
**Cloud only:**

`Cloud only` runs against Ollama's hosted API at `https://ollama.com`.

    Use **Cloud only** during setup. OpenClaw prompts for `OLLAMA_API_KEY`, sets `baseUrl: "https://ollama.com"`, and seeds the hosted cloud model list. This path does **not** require a local Ollama server or `ollama signin`.

    The cloud model list shown during `openclaw onboard` is populated live from `https://ollama.com/api/tags`, capped at 500 entries, so the picker reflects the current hosted catalog rather than a static seed. If `ollama.com` is unreachable or returns no models at setup time, OpenClaw falls back to the previous hardcoded suggestions so onboarding still completes.

  
**Local only:**

In local-only mode, OpenClaw discovers models from the configured Ollama instance. This path is for local or self-hosted Ollama servers.

    OpenClaw currently suggests `gemma4` as the local default.

  
## Model discovery (implicit provider)

When you set `OLLAMA_API_KEY` (or an auth profile) and **do not** define `models.providers.ollama` or another custom remote provider with `api: "ollama"`, OpenClaw discovers models from the local Ollama instance at `http://127.0.0.1:11434`.

| Behavior             | Detail                                                                                                                                                               |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Catalog query        | Queries `/api/tags`                                                                                                                                                  |
| Capability detection | Uses best-effort `/api/show` lookups to read `contextWindow`, expanded `num_ctx` Modelfile parameters, and capabilities including vision/tools                       |
| Vision models        | Models with a `vision` capability reported by `/api/show` are marked as image-capable (`input: ["text", "image"]`), so OpenClaw auto-injects images into the prompt  |
| Reasoning detection  | Uses `/api/show` capabilities when available, including `thinking`; falls back to a model-name heuristic (`r1`, `reasoning`, `think`) when Ollama omits capabilities |
| Token limits         | Sets `maxTokens` to the default Ollama max-token cap used by OpenClaw                                                                                                |
| Costs                | Sets all costs to `0`                                                                                                                                                |

This avoids manual model entries while keeping the catalog aligned with the local Ollama instance. You can use a full ref such as `ollama/<pulled-model>:latest` in local `infer model run`; OpenClaw resolves that installed model from Ollama's live catalog without requiring a hand-written `models.json` entry.

For signed-in Ollama hosts, some `:cloud` models may be usable through `/api/chat`
and `/api/show` before they appear in `/api/tags`. When you explicitly select a
full `ollama/<model>:cloud` ref, OpenClaw validates that exact missing model with
`/api/show` and adds it to the runtime catalog only if Ollama confirms model
metadata. Typos still fail as unknown models instead of being auto-created.

```bash
# See what models are available
ollama list
openclaw models list
```

For a narrow text-generation smoke test that avoids the full agent tool surface,
use local `infer model run` with a full Ollama model ref:

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "Reply with exactly: pong" \
    --json
```

That path still uses OpenClaw's configured provider, auth, and native Ollama
transport, but it does not start a chat-agent turn or load MCP/tool context. If
this succeeds while normal agent replies fail, troubleshoot the model's agent
prompt/tool capacity next.

For a narrow vision-model smoke test on the same lean path, add one or more
image files to `infer model run`. This sends the prompt and image directly to
the selected Ollama vision model without loading chat tools, memory, or prior
session context:

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "Describe this image in one sentence." \
    --file ./photo.jpg \
    --json
```

`model run --file` accepts files detected as `image/*`, including common PNG,
JPEG, and WebP inputs. Non-image files are rejected before Ollama is called.
For speech recognition, use `openclaw infer audio transcribe` instead.

When you switch a conversation with `/model ollama/<model>`, OpenClaw treats
that as an exact user selection. If the configured Ollama `baseUrl` is
unreachable, the next reply fails with the provider error instead of silently
answering from another configured fallback model.

Isolated cron jobs do one extra local safety check before they start the agent
turn. If the selected model resolves to a local, private-network, or `.local`
Ollama provider and `/api/tags` is unreachable, OpenClaw records that cron run
as `skipped` with the selected `ollama/<model>` in the error text. The endpoint
preflight is cached for 5 minutes, so multiple cron jobs pointed at the same
stopped Ollama daemon do not all launch failing model requests.

Live-verify the local text path, native stream path, and embeddings against
local Ollama with:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

To add a new model, simply pull it with Ollama:

```bash
ollama pull mistral
```

The new model will be automatically discovered and available to use.

> **Note:** If you set `models.providers.ollama` explicitly, or configure a custom remote provider such as `models.providers.ollama-cloud` with `api: "ollama"`, auto-discovery is skipped and you must define models manually. Loopback custom providers such as `http://127.0.0.2:11434` are still treated as local. See the explicit config section below.


## Vision and image description

The bundled Ollama plugin registers Ollama as an image-capable media-understanding provider. This lets OpenClaw route explicit image-description requests and configured image-model defaults through local or hosted Ollama vision models.

For local vision, pull a model that supports images:

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
```

Then verify with the infer CLI:

```bash
openclaw infer image describe \
  --file ./photo.jpg \
  --model ollama/qwen2.5vl:7b \
  --json
```

`--model` must be a full `<provider/model>` ref. When it is set, `openclaw infer image describe` runs that model directly instead of skipping description because the model supports native vision.

Use `infer image describe` when you want OpenClaw's image-understanding provider flow, configured `agents.defaults.imageModel`, and image-description output shape. Use `infer model run --file` when you want a raw multimodal model probe with a custom prompt and one or more images.

To make Ollama the default image-understanding model for inbound media, configure `agents.defaults.imageModel`:

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

Prefer the full `ollama/<model>` ref. If the same model is listed under `models.providers.ollama.models` with `input: ["text", "image"]` and no other configured image provider exposes that bare model ID, OpenClaw also normalizes a bare `imageModel` ref such as `qwen2.5vl:7b` to `ollama/qwen2.5vl:7b`. If more than one configured image provider 
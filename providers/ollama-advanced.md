---
domain: providers
topic: "Ollama Advanced: Cloud Models, Vision, Web Search, and Troubleshooting"
type: reference
keywords:
  - Ollama vision
  - Ollama web search
  - Ollama troubleshooting
  - Ollama cloud
  - Ollama config
related:
  - providers/ollama
  - gateway/local-models
source: providers/ollama.md
---

Ollama advanced configuration, troubleshooting, and common recipes.

: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```

## Cloud models

    `Cloud + Local` uses a reachable Ollama host as the control point for both local and cloud models. This is Ollama's preferred hybrid flow.

    Use **Cloud + Local** during setup. OpenClaw prompts for the Ollama base URL, discovers local models from that host, and checks whether the host is signed in for cloud access with `ollama signin`. When the host is signed in, OpenClaw also suggests hosted cloud defaults such as `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, and `glm-5.1:cloud`.

    If the host is not signed in yet, OpenClaw keeps the setup local-only until you run `ollama signin`.

    `Cloud only` runs against Ollama's hosted API at `https://ollama.com`.

    Use **Cloud only** during setup. OpenClaw prompts for `OLLAMA_API_KEY`, sets `baseUrl: "https://ollama.com"`, and seeds the hosted cloud model list. This path does **not** require a local Ollama server or `ollama signin`.

    The cloud model list shown during `openclaw onboard` is populated live from `https://ollama.com/api/tags`, capped at 500 entries, so the picker reflects the current hosted catalog rather than a static seed. If `ollama.com` is unreachable or returns no models at setup time, OpenClaw falls back to the previous hardcoded suggestions so onboarding still completes.

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

If you set `models.providers.ollama` explicitly, or configure a custom remote provider such as `models.providers.ollama-cloud` with `api: "ollama"`, auto-discovery is skipped and you must define models manually. Loopback custom providers such as `http://127.0.0.2:11434` are still treated as local. See the explicit config section below.

## Vision and image description

The bundled Ollama plugin r

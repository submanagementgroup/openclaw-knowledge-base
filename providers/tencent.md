---
domain: providers
topic: "Tencent Cloud TokenHub Provider: Hy3 Preview Setup, Models, and Pricing"
type: integration
keywords:
  - tencent
  - tencent-tokenhub
  - TokenHub
  - hy3-preview
  - TOKENHUB_API_KEY
  - Hunyuan
  - Tencent Cloud
  - tokenhub-api-key
  - tencent provider
  - reasoning model
source: providers/tencent.md
related:
  - concepts/model-providers
  - gateway/configuration-reference
  - providers/overview
---

Tencent Cloud ships as a bundled provider plugin in OpenClaw, giving access to Tencent Hy3 preview through the TokenHub endpoint (`tencent-tokenhub`) using an OpenAI-compatible API. Hy3 preview is Tencent Hunyuan's large MoE language model for reasoning, long-context instruction following, code, and agent workflows.

| Property | Value |
|---|---|
| Provider id | `tencent-tokenhub` |
| Plugin | bundled, `enabledByDefault: true` |
| Auth env var | `TOKENHUB_API_KEY` |
| Onboarding flag | `--auth-choice tokenhub-api-key` |
| Direct CLI flag | `--tokenhub-api-key <key>` |
| API | OpenAI-compatible (`openai-completions`) |
| Default base URL | `https://tokenhub.tencentmaas.com/v1` |
| Global base URL | `https://tokenhub-intl.tencentmaas.com/v1` (override) |
| Default model | `tencent-tokenhub/hy3-preview` |

## Quick Start

### Step 1: Create a TokenHub API Key

Create an API key in Tencent Cloud TokenHub. If you choose a limited access scope for the key, include **Hy3 preview** in the allowed models.

### Step 2: Run Onboarding

```bash
# Interactive onboarding
openclaw onboard --auth-choice tokenhub-api-key

# Direct flag
openclaw onboard --non-interactive \
  --auth-choice tokenhub-api-key \
  --tokenhub-api-key "$TOKENHUB_API_KEY"

# Env only
export TOKENHUB_API_KEY=...
```

### Step 3: Verify the Model

```bash
openclaw models list --provider tencent-tokenhub
```

## Non-Interactive Setup

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice tokenhub-api-key \
  --tokenhub-api-key "$TOKENHUB_API_KEY" \
  --skip-health \
  --accept-risk
```

## Built-In Catalog

| Model ref | Name | Input | Context | Max output | Notes |
|---|---|---|---|---|---|
| `tencent-tokenhub/hy3-preview` | Hy3 preview (TokenHub) | text | 256,000 | 64,000 | Default; reasoning-enabled |

The model id is `hy3-preview`. Do not confuse it with Tencent's `HY-3D-*` models (3D generation APIs). Tencent's OpenAI-compatible examples use `hy3-preview` as the model id and support standard chat-completions tool calling plus `reasoning_effort`.

## Tiered Pricing

The bundled catalog ships tiered cost metadata that scales with input window length.

| Input tokens range | Input rate | Output rate | Cache read |
|---|---|---|---|
| 0 – 16,000 | 0.176 | 0.587 | 0.059 |
| 16,000 – 32,000 | 0.235 | 0.939 | 0.088 |
| 32,000+ | 0.293 | 1.173 | 0.117 |

Rates are per million tokens in USD as advertised by Tencent. Override pricing under `models.providers.tencent-tokenhub` only when you need a different surface.

## Advanced Configuration

### Endpoint Override

OpenClaw defaults to `https://tokenhub.tencentmaas.com/v1`. Override for the international endpoint:

```bash
openclaw config set models.providers.tencent-tokenhub.baseUrl "https://tokenhub-intl.tencentmaas.com/v1"
```

Only override when your TokenHub account or region requires it.

### Environment Availability for the Daemon

If the Gateway runs as a managed service (launchd, systemd, Docker), `TOKENHUB_API_KEY` must be visible to that process. Set it in `~/.openclaw/.env` or via `env.shellEnv` so launchd, systemd, or Docker exec environments can read it.

Keys set only in `~/.profile` are not visible to managed gateway processes.

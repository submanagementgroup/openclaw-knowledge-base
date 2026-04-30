---
domain: reference
topic: "Prompt Caching: Reducing API Costs by Caching Prompt Prefixes"
type: reference
keywords:
  - prompt caching
  - cache prefix
  - API cost reduction
  - Anthropic caching
  - cache tokens
related:
  - providers/anthropic
  - reference/token-use
source: reference/prompt-caching.md
---

Prompt caching reduces costs by caching repeated prompt prefixes between calls. Supported by Anthropic and OpenAI.

Prompt caching means the model provider can reuse unchanged prompt prefixes (usually system/developer instructions and other stable context) across turns instead of re-processing them every time. OpenClaw normalizes provider usage into `cacheRead` and `cacheWrite` where the upstream API exposes those counters directly.

Status surfaces can also recover cache counters from the most recent transcript
usage log when the live session snapshot is missing them, so `/status` can keep
showing a cache line after partial session metadata loss. Existing nonzero live
cache values still take precedence over transcript fallback values.

Why this matters: lower token cost, faster responses, and more predictable performance for long-running sessions. Without caching, repeated prompts pay the full prompt cost on every turn even when most input did not change.

The sections below cover every cache-related knob that affects prompt reuse and token cost.

Provider references:

- Anthropic prompt caching: [https://platform.claude.com/docs/en/build-with-claude/prompt-caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- OpenAI prompt caching: [https://developers.openai.com/api/docs/guides/prompt-caching](https://developers.openai.com/api/docs/guides/prompt-caching)
- OpenAI API headers and request IDs: [https://developers.openai.com/api/reference/overview](https://developers.openai.com/api/reference/overview)
- Anthropic request IDs and errors: [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

## Primary knobs

### `cacheRetention` (global default, model, and per-agent)

Set cache retention as a global default for all models:

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
```

Override per-model:

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # none | short | long
```

Per-agent override:

```yaml
agents:
  list:
    - id: "alerts"
      params:
        cacheRetention: "none"
```

Config merge order:

1. `agents.defaults.params` (global default — applies to all models)
2. `agents.defaults.models["provider/model"].params` (per-model override)
3. `agents.list[].params` (matching agent id; overrides by key)

### `contextPruning.mode: "cache-ttl"`

Prunes old tool-result context after cache TTL windows so post-idle requests do not re-cache oversized history.

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

See [Session Pruning](/concepts/session-pruning) for full behavior.

### Heartbeat keep-warm

Heartbeat can keep cache windows warm and reduce repeated cache writes after idle gaps.

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

Per-agent heartbeat is supported at `agents.list[].heartbeat`.

## Provider behavior

### Anthropic (direct API)

- `cacheRetention` is supported.
- With Anthropic API-key auth profiles, OpenClaw seeds `cacheRetention: "short"` for Anthropic model refs when unset.
- Anthropic native Messages responses expose both `cache_read_input_tokens` and `cache_creation_input_tokens`, so OpenClaw can show both `cacheRead` and `cacheWrite`.
- For native Anthropic requests, `cacheRetention: "short"` maps to the default 5-minute ephemeral cache, and `cacheRetention: "long"` upgrades to the 1-hour TTL only on direct `api.anthropic.com` hosts.

### OpenAI (direct API)

- Prompt caching is automatic on supported recent models. OpenClaw does not need to inject block-level cache markers.
- OpenClaw uses `prompt_cache_key` to keep cache routing stable across turns and uses `prompt_cache_retention: "24h"` only when `cacheRetention: "long"` is selected on direct OpenAI hosts.
- OpenAI-compatible Completions providers receive `prompt_cache_key` only when their model config explicitly sets `compat.supportsPromptCacheKey: true`; `cacheRetention: "none"` still suppresses it.
- OpenAI responses expose cached prompt tokens via `usage.prompt_tokens_details.cached_tokens` (or `input_tokens_details.cached_tokens` on Responses API events). OpenClaw maps that to `cacheRead`.
- OpenAI does not expose a separate cache-write token counter, so `cacheWrite` stays `0` on OpenAI paths even when the provider is warming a cache.
- OpenAI returns useful tracing and rate-limit headers such as `x-request-id`, `openai-processing-ms`, and `x-ratelimit-*`, but cache-hit accounting should come from the usage payload, not from headers.
- In practice, OpenAI often behaves like an initial-prefix cache rather than Anthropic-style moving full-history reuse. Stable long-prefix text turns can land near a `4864` cached-token plateau in current live probes, while tool-heavy or MCP-style transcripts often plateau near `4608` cached tokens even on exact repeats.

### Anthropic Vertex

- Anthropic models on Vertex AI (`anthropic-vertex/*`) support `cacheRetention` the same way as direct Anthropic.
- `cacheRetention: "long"` maps to the real 1-hour prompt-cache TTL on Vertex AI endpoints.
- Default cache retention for `anthropic-vertex` matches direct Anthropic defaults.
- Vertex requests are routed through boundary-aware cache shaping so cache reuse stays aligned with what providers actually receive.

### Amazon Bedrock

- Anthropic Claude model refs (`amazon-bedrock/*anthropic.claude*`) support explicit `cacheRetention` pass-through.
- Non-Anthropic Bedrock models are forced to `cacheRetention: "none"` at runtime.

### OpenRouter models

For `openrouter/anthropic/*` model refs, OpenClaw injects Anthropic
`cache_control` on system/developer prompt blocks to improve prompt-cache
reuse only when the request is still targeting a verified OpenRouter route
(`openrouter` on its default endpoint, or any provider/base URL that resolves
to `openrouter.ai`).

For `openrouter/deepseek/*`, `openrouter/moonshot*/*`, and `openrouter/zai/*`
model refs, `contextPruning.mode: "cache-ttl"` is allowed because OpenRouter
handles provider-side prompt caching automatically. OpenClaw does not inject
Anthropic `cache_control` markers into those requests.

DeepSeek cache construction is best-effort and can take a few seconds. An
immediate follow-up may still show `cached_tokens: 0`; verify with a repeated
same-prefix request after a short delay and use `usage.prompt_tokens_details.cached_tokens`
as the cache-hit signal.

If you repoint the model at an arbitrary OpenAI-compatible proxy URL, OpenClaw
stops injecting those OpenRouter-specific Anthropic cache markers.

### Other providers

If the provider does not support this cache mode, `cacheRetention` has no effect.

### Google Gemini direct API

- Direct Gemini transport (`api: "google-generative-ai"`) reports cache hits
  through upstream `cachedContentTokenCount`; OpenClaw maps that to `cacheRead`.
- When `cacheRetention` is set on a direct Gemini model, OpenClaw automatically
  creates, reuses, and refreshes `cachedContents` resources for system prompts
  on Google AI Studio runs. This means you no longer need to pre-create a
  cached-content handle manually.
- You can still pass a pre-existing Gemini cached-content handle through as
  `params.cachedContent` (or legacy `params.cached_content`) on the configured
  model.
- This is separate from Anthropic/OpenAI prompt-prefix caching. For Gemini,
  OpenClaw manages a provider-native `cachedContents` resource rather than
  injecting cache markers into the request.

### Gemini CLI JSON usage

- Gemini CLI JSON output can also surface cache hits through `stats.cached`;
  OpenClaw maps that to `cacheRead`.
- If the CLI omits a direct `stats.input` value, OpenClaw derives input tokens
  from `stats.input_tokens - stats.cached`.
- This is usage normalization only. It does not mean OpenClaw is creating
  Anthropic/OpenAI-style prompt-cache markers for Gemini CLI.

## System-prompt cache boundary

OpenClaw splits the system prompt into a **stable prefix** and a **volatile
suffix*

---
domain: concepts
topic: "Model Provider System: Configuration and Selection"
type: concept
keywords:
  - model providers
  - provider config
  - model selection
  - models.providers
  - provider API
  - model registry
source: concepts/model-providers.md
---

Reference for **LLM/model providers** (not chat channels like WhatsApp/Telegram). For model selection rules, see [Models](/concepts/models).

## Quick rules

### Model refs and CLI helpers

- Model refs use `provider/model` (example: `opencode/claude-opus-4-6`).
    - `agents.defaults.models` acts as an allowlist when set.
    - CLI helpers: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
    - `models.providers.*.contextWindow` / `contextTokens` / `maxTokens` set provider-level defaults; `models.providers.*.models[].contextWindow` / `contextTokens` / `maxTokens` override them per model.
    - Fallback rules, cooldown probes, and session-override persistence: [Model failover](/concepts/model-failover).

  ### Adding provider auth does not change your primary model

`openclaw configure` preserves an existing `agents.defaults.model.primary` when you add or reauth a provider. `openclaw models auth login` does the same unless you pass `--set-default`. Provider plugins may still return a recommended default model in their auth config patch, but OpenClaw treats that as "make this model available" when a primary model already exists, not "replace the current primary model."

    To intentionally switch the default model, use `openclaw models set <provider/model>` or `openclaw models auth login --provider <id> --set-default`.

  ### OpenAI provider/runtime split

OpenAI-family routes are prefix-specific:

    - `openai/<model>` uses the native Codex app-server harness for agent turns by default. This is the usual ChatGPT/Codex subscription setup.
    - `openai-codex/<model>` is legacy config that doctor rewrites to `openai/<model>`.
    - `openai/<model>` plus provider/model `agentRuntime.id: "pi"` uses PI for explicit API-key or compatibility routes.

    See [OpenAI](/providers/openai) and [Codex harness](/plugins/codex-harness). If the provider/runtime split is confusing, read [Agent runtimes](/concepts/agent-runtimes) first.

    Plugin auto-enable follows the same boundary: `openai/*` agent refs enable the Codex plugin for the default route, and explicit provider/model `agentRuntime.id: "codex"` or legacy `codex/<model>` refs also require it.

    GPT-5.5 is available through the native Codex app-server harness by default on `openai/gpt-5.5`, and through PI only when provider/model runtime policy explicitly selects `pi`.

  ### CLI runtimes

CLI runtimes use the same split: choose canonical model refs such as `anthropic/claude-*`, `google/gemini-*`, or `openai/gpt-*`, then set provider/model runtime policy to `claude-cli`, `google-gemini-cli`, or `codex-cli` when you want a local CLI backend.

    Legacy `claude-cli/*`, `google-gemini-cli/*`, and `codex-cli/*` refs migrate back to canonical provider refs with the runtime recorded separately.

  ## Plugin-owned provider behavior

Most provider-specific logic lives in provider plugins (`registerProvider(...)`) while OpenClaw keeps the generic inference loop. Plugins own onboarding, model catalogs, auth env-var mapping, transport/config normalization, tool-schema cleanup, failover classification, OAuth refresh, usage reporting, thinking/reasoning profiles, and more.

The full list of provider-SDK hooks and bundled-plugin examples lives in [Provider plugins](/plugins/sdk-provider-plugins). A provider that needs a totally custom request executor is a separate, deeper extension surface.

> **Note:** Provider-owned runner behavior lives on explicit provider hooks such as replay policy, tool-schema normalization, stream wrapping, and transport/request helpers. The legacy `ProviderPlugin.capabilities` static bag is compatibility-only and is no longer read by shared runner logic.


## API key rotation

### Key sources and priority

Configure multiple keys via:

    - `OPENCLAW_LIVE__KEY` (single live override, highest priority)
    - `_API_KEYS` (comma or semicolon list)
    - `_API_KEY` (primary key)
    - `_API_KEY_*` (numbered list, e.g. `_API_KEY_1`)

    For Google providers, `GOOGLE_API_KEY` is also included as fallback. Key selection order preserves priority and deduplicates values.

  ### When rotation kicks in

- Requests are retried with the next key only on rate-limit responses (for example `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, or periodic usage-limit messages).
    - Non-rate-limit failures fail immediately; no key rotation is attempted.
    - When all candidate keys fail, the final error is returned from the last attempt.

  ## Built-in providers (pi-ai catalog)

OpenClaw ships with the pi-ai catalog. These providers require **no** `models.providers` config; just set auth + pick a model.

### OpenAI

- Provider: `openai`
- Auth: `OPENAI_API_KEY`
- Optional rotation: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, plus `OPENCLAW_LIVE_OPENAI_KEY` (single override)
- Example models: `openai/gpt-5.5`, `openai/gpt-5.4-mini`
- Verify account/model availability with `openclaw models list --provider openai` if a specific install or API key behaves differently.
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Default transport is `auto`; OpenClaw passes the transport choice to pi-ai.
- Override per model via `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"`, or `"auto"`)
- OpenAI priority processing can be enabled via `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` and `params.fastMode` map direct `openai/*` Responses requests to `service_tier=priority` on `api.openai.com`
- Use `params.serviceTier` when you want an explicit tier instead of the shared `/fast` toggle
- Hidden OpenClaw attribution headers (`originator`, `version`, `User-Agent`) apply only on native OpenAI traffic to `api.openai.com`, not generic OpenAI-compatible proxies
- Native OpenAI routes also keep Responses `store`, prompt-cache hints, and OpenAI reasoning-compat payload shaping; proxy routes do not
- `openai/gpt-5.3-codex-spark` is intentionally suppressed in OpenClaw because live OpenAI API requests reject it and the current Codex catalog does not expose it

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.5" } } },
}
```

### Anthropic

- Provider: `anthropic`
- Auth: `ANTHROPIC_API_KEY`
- Optional rotation: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, plus `OPENCLAW_LIVE_ANTHROPIC_KEY` (single override)
- Example model: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Direct public Anthropic requests support the shared `/fast` toggle and `params.fastMode`, including API-key and OAuth-authenticated traffic sent to `api.anthropic.com`; OpenClaw maps that to Anthropic `service_tier` (`auto` vs `standard_only`)
- Preferred Claude CLI config keeps the model ref canonical and selects the CLI
  backend separately: `anthropic/claude-opus-4-7` with
  model-scoped `agentRuntime.id: "claude-cli"`. Legacy
  `claude-cli/claude-opus-4-7` refs still work for compatibility.

> **Note:** Anthropic staff told us OpenClaw-style Claude CLI usage is allowed again, so OpenClaw treats Claude CLI reuse and `claude -p` usage as sanctioned for this integration unless Anthropic publishes a new policy. Anthropic setup-token remains available as a supported OpenClaw token path, but OpenClaw now prefers Claude CLI reuse and `claude -p` when available.


```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Codex OAuth

- Provider: `openai-codex`
- Auth: OAuth (ChatGPT)
- Legacy PI model ref: `openai-codex/gpt-5.5`
- Native Codex app-server harness ref: `openai/gpt-5.5`
- Native Codex app-server harness docs: [Codex harness](/plugins/codex-harness)
- Legacy model refs: `codex/gpt-*`
- Plugin boundary: `openai-codex/*` loads the OpenAI plugin; the native Codex app-server plugin is selected only by the Codex harness runtime or legacy `codex/*` refs.
- CLI: `openclaw onboard --auth-choice openai-codex` or `openclaw models auth login --provider openai-codex`
- Default transport is `auto` (WebSocket-first, SSE fallback)
- Override per PI model via `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"`, or `"auto"`)
- `params.serviceTier` is also forwarded on native Codex Responses requests (`chatgpt.com/backend-api`)
- Hidden OpenClaw attribution headers (`originator`, `version`, `User-Agent`) are only attached on native Codex traffic to `chatgpt.com/backend-api`, not generic OpenAI-compatible proxies
- Shares the same `/fast` toggle and `params.fastMode` config as direct `openai/*`; OpenClaw maps that to `service_tier=priority`
- `openai-codex/gpt-5.5` uses the Codex catalog native `contextWindow = 400000` and default runtime `contextTokens = 272000`; override the runtime cap with `models.providers.openai-codex.models[].contextTokens`
- Policy note: OpenAI Codex OAuth is explicitly supported for external tools/workflows like OpenClaw.
- For the common subscription plus native Codex runtime route, sign in with `openai-codex` auth but configure `openai/gpt-5.5`; OpenAI agent turns select Codex by default.
- Use provider/model `agentRuntime.id: "pi"` only when you want a compatibility route through PI; otherwise keep `openai/gpt-5.5` on the default Codex harness.
- Older `openai-codex/gpt-5.1*`, `openai-codex/gpt-5.2*`, and `openai-codex/gpt-5.3*` refs are suppressed because ChatGPT/Codex OAuth accounts reject them; use `openai-codex/gpt-5.5` or the native Codex runtime route instead.

```json5
{
  plugins: { entries: { codex: { enabled: true } } },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.5" },
    },
  },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.5", contextTokens: 160000 }],
      },
    },
  },
}
```

### Other subscription-style hosted options

**GLM models:** Z.AI Coding Plan or general API endpoints.
  

**MiniMax:** MiniMax Coding Plan OAuth or API key access.
  

**Qwen Cloud:** Qwen Cloud provider surface plus Alibaba DashScope and Coding Plan endpoint mapping.
  

### OpenCode

- Auth: `OPENCODE_API_KEY` (or `OPENCODE_ZEN_API_KEY`)
- Zen runtime provider: `opencode`
- Go runtime provider: `opencode-go`
- Example models: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice opencode-zen` or `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API key)

- Provider: `google`
- Auth: `GEMINI_API_KEY`
- Optional rotation: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, `GOOGLE_API_KEY` fallback, and `OPENCLAW_LIVE_GEMINI_KEY` (single override)
- Example models: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Compatibility: legacy OpenClaw config using `google/gemini-3.1-flash-preview` is normalized to `google/gemini-3-flash-preview`
- Alias: `google/gemini-3.1-pro` is accepted and normalized to Google's live Gemini API id, `google/gemini-3.1-pro-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Thinking: `/think adaptive` uses Google dynamic thinking. Gemini 3/3.1 omit a fixed `thinkingLevel`; Gemini 2.5 sends `thinkingBudget: -1`.
- Direct Gemini runs also accept `agents.defaults.models["google/<model>"].params.cachedContent` (or legacy `cached_content`) to forward a provider-native `cachedContents/...` handle; Gemini cache hits surface as OpenClaw `cacheRead`

### Google Vertex and Gemini CLI

- Providers: `google-vertex`, `google-gemini-cli`
- Auth: Vertex uses gcloud ADC; Gemini CLI uses its OAuth flow

> **Note:** Gemini CLI OAuth in OpenClaw is an unofficial integration. Some users have reported Google account restrictions after using third-party clients. Review Google terms and use a non-critical account if you choose to proceed.


Gemini CLI OAuth is shipped as part of the bundled `google` plugin.

**Install Gemini CLI**

**brew:**

```bash
        brew install gemini-cli
        ```
      
**npm:**

```bash
        npm install -g @google/gemini-cli
        ```
      

**Enable plugin**

```bash
    openclaw plugins enable google
    ```
  
**Login**

```bash
    openclaw models auth login --provider google-gemini-cli --set-default
    ```

    Default model: `google-gemini-cli/gemini-3-flash-preview`. You do **not** paste a client id or secret into `openclaw.json`. The CLI login flow stores tokens in auth profiles on the gateway host.

  
**Set project (if needed)**

If requests fail after login, set `GOOGLE_CLOUD_PROJECT` or `GOOGLE_CLOUD_PROJECT_ID` on the gateway host.
  
Gemini CLI JSON replies are parsed from `response`; usage falls back to `stats`, with `stats.cached` normalized into OpenClaw `cacheRead`.

### Z.AI (GLM)

- Provider: `zai`
- Auth: `ZAI_API_KEY`
- Example model: `zai/glm-5.1`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Aliases: `z.ai/*` and `z-ai/*` normalize to `zai/*`
  - `zai-api-key` auto-detects the matching Z.AI endpoint; `zai-coding-global`, `zai-coding-cn`, `zai-global`, and `zai-cn` force a specific surface

### Vercel AI Gateway

- Provider: `vercel-ai-gateway`
- Auth: `AI_GATEWAY_API_KEY`
- Example models: `vercel-ai-gateway/anthropic/claude-opus-4.6`, `vercel-ai-gateway/moonshotai/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Provider: `kilocode`
- Auth: `KILOCODE_API_KEY`
- Example model: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- Base URL: `https://api.kilo.ai/api/gateway/`
- Static fallback catalog ships `kilocode/kilo/auto`; live `https://api.kilo.ai/api/gateway/models` discovery can expand the runtime catalog further.
- Exact upstream routing behind `kilocode/kilo/auto` is owned by Kilo Gateway, not hard-coded in OpenClaw.

See [/providers/k
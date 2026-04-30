---
domain: reference
topic: "API Usage Costs: Cost Tracking, Reporting, and Provider Pricing Reference"
type: reference
keywords:
  - API costs
  - usage costs
  - cost tracking
  - provider pricing
  - billing
related:
  - reference/token-use
  - reference/prompt-caching
source: reference/api-usage-costs.md
---

API usage cost tracking and reporting in OpenClaw.

# API usage & costs

This doc lists **features that can invoke API keys** and where their costs show up. It focuses on
OpenClaw features that can generate provider usage or paid API calls.

## Where costs show up (chat + CLI)

**Per-session cost snapshot**

- `/status` shows the current session model, context usage, and last response tokens.
- If the model uses **API-key auth**, `/status` also shows **estimated cost** for the last reply.
- If live session metadata is sparse, `/status` can recover token/cache
  counters and the active runtime model label from the latest transcript usage
  entry. Existing nonzero live values still take precedence, and prompt-sized
  transcript totals can win when stored totals are missing or smaller.

**Per-message cost footer**

- `/usage full` appends a usage footer to every reply, including **estimated cost** (API-key only).
- `/usage tokens` shows tokens only; subscription-style OAuth/token and CLI flows hide dollar cost.
- Gemini CLI note: when the CLI returns JSON output, OpenClaw reads usage from
  `stats`, normalizes `stats.cached` into `cacheRead`, and derives input tokens
  from `stats.input_tokens - stats.cached` when needed.

Anthropic note: Anthropic staff told us OpenClaw-style Claude CLI usage is
allowed again, so OpenClaw treats Claude CLI reuse and `claude -p` usage as
sanctioned for this integration unless Anthropic publishes a new policy.
Anthropic still does not expose a per-message dollar estimate that OpenClaw can
show in `/usage full`.

**CLI usage windows (provider quotas)**

- `openclaw status --usage` and `openclaw channels list` show provider **usage windows**
  (quota snapshots, not per-message costs).
- Human output is normalized to `X% left` across providers.
- Current usage-window providers: Anthropic, GitHub Copilot, Gemini CLI,
  OpenAI Codex, MiniMax, Xiaomi, and z.ai.
- MiniMax note: its raw `usage_percent` / `usagePercent` fields mean remaining
  quota, so OpenClaw inverts them before display. Count-based fields still win
  when present. If the provider returns `model_remains`, OpenClaw prefers the
  chat-model entry, derives the window label from timestamps when needed, and
  includes the model name in the plan label.
- Usage auth for those quota windows comes from provider-specific hooks when
  available; otherwise OpenClaw falls back to matching OAuth/API-key
  credentials from auth profiles, env, or config.

See [Token use & costs](/reference/token-use) for details and examples.

## How keys are discovered

OpenClaw can pick up credentials from:

- **Auth profiles** (per-agent, stored in `auth-profiles.json`).
- **Environment variables** (e.g. `OPENAI_API_KEY`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`).
- **Config** (`models.providers.*.apiKey`, `plugins.entries.*.config.webSearch.apiKey`,
  `plugins.entries.firecrawl.config.webFetch.apiKey`, `memorySearch.*`,
  `talk.providers.*.apiKey`).
- **Skills** (`skills.entries.<name>.apiKey`) which may export keys to the skill process env.

## Features that can spend keys

### 1) Core model responses (chat + tools)

Every reply or tool call uses the **current model provider** (OpenAI, Anthropic, etc). This is the
primary source of usage and cost.

This also includes subscription-style hosted providers that still bill outside
OpenClaw's local UI, such as **OpenAI Codex**, **Alibaba Cloud Model Studio
Coding Plan**, **MiniMax Coding Plan**, **Z.AI / GLM Coding Plan**, and
Anthropic's OpenClaw Claude-login path with **Extra Usage** enabled.

See [Models](/providers/models) for pricing config and [Token use & costs](/reference/token-use) for display.

### 2) Media understanding (audio/image/video)

Inbound media can be summarized/transcribed before the reply runs. This uses model/provider APIs.

- Audio: OpenAI / Groq / Deepgram / DeepInfra / Google / Mistral.
- Image: OpenAI / OpenRouter / Anthropic / DeepInfra / Google / MiniMax / Moonshot / Qwen / Z.AI.
- Video: Google / Qwen / Moonshot.

See [Media understanding](/nodes/media-understanding).

### 3) Image and video generation

Shared generation capabilities can also spend provider keys:

- Image generation: OpenAI / Google / DeepInfra / fal / MiniMax
- Video generation: DeepInfra / Qwen

Image generation can infer an auth-backed provider default when
`agents.defaults.imageGenerationModel` is unset. Video generation currently
requires an explicit `agents.defaults.videoGenerationModel` such as
`qwen/wan2.6-t2v`.

See [Image generation](/tools/image-generation), [Qwen Cloud](/providers/qwen),
and [Models](/concepts/models).

### 4) Memory embeddings + semantic search

Semantic memory search uses **embedding APIs** when configured for remote providers:

- `memorySearch.provider = "openai"` → OpenAI embeddings
- `memorySearch.provider = "gemini"` → Gemini embeddings
- `memorySearch.provider = "voyage"` → Voyage embeddings
- `memorySearch.provider = "mistral"` → Mistral embeddings
- `memorySearch.provider = "deepinfra"` → DeepInfra embeddings
- `memorySearch.provider = "lmstudio"` → LM Studio embeddings (local/self-hosted)
- `memorySearch.provider = "ollama"` → Ollama embeddings (local/self-hosted; typically no hosted API billing)
- Optional fallback to a remote provider if local embeddings fail

You can keep it local with `memorySearch.provider = "local"` (no API usage).

See [Memory](/concepts/memory).

### 5) Web search tool

`web_search` may incur usage charges depending on your provider:

- **Brave Search API**: `BRAVE_API_KEY` or `plugins.entries.brave.config.webSearch.apiKey`
- **Exa**: `EXA_API_KEY` or `plugins.entries.exa.config.webSearch.apiKey`
- **Firecrawl**: `FIRECRAWL_API_KEY` or `plugins.entries.firecrawl.config.webSearch.apiKey`
- **Gemini (Google Search)**: `GEMINI_API_KEY` or `plugins.entries.google.config.webSearch.apiKey`
- **Grok (xAI)**: `XAI_API_KEY` or `plugins.entries.xai.config.webSearch.apiKey`
- **Kimi (Moonshot)**: `KIMI_API_KEY`, `MOONSHOT_API_KEY`, or `plugins.entries.moonshot.config.webSearch.apiKey`
- **MiniMax Search**: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_API_KEY`, or `plugins.entries.minimax.config.webSearch.apiKey`
- **Ollama Web Search**: key-free for a reachable signed-in local Ollama host; direct `https://ollama.com` search uses `OLLAMA_API_KEY`, and auth-protected hosts can reuse normal Ollama provider bearer auth
- **Perplexity Search API**: `PERPLEXITY_API_KEY`, `OPENROUTER_API_KEY`, or `plugins.entries.perplexity.config.webSearch.apiKey`
- **Tavily**: `TAVILY_API_KEY` or `plugins.entries.tavily.config.webSearch.apiKey`
- **DuckDuckGo**: key-free fallback (no API billing, but unofficial and HTML-based)
- **SearXNG**: `SEARXNG_BASE_URL` or `plugins.entries.searxng.config.webSearch.baseUrl` (key-free/self-hosted; no hosted API billing)

Legacy `tools.web.search.*` provider paths still load through the temporary compatibility shim, but they are no longer the recommended config surface.

**Brave Search free credit:** Each Brave plan includes \$5/month i

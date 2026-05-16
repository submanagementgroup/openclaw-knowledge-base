---
domain: reference
topic: "API Usage Costs, Tokens, and Credits"
type: reference
keywords:
  - API costs
  - token usage
  - credits
  - billing
  - usage tracking
  - cost estimation
source: 
  - reference/api-usage-costs.md
  - reference/token-use.md
  - reference/credits.md
---

## API Usage Costs

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

**Brave Search free credit:** Each Brave plan includes \$5/month in renewing
free credit. The Search plan costs \$5 per 1,000 requests, so the credit covers
1,000 requests/month at no charge. Set your usage limit in the Brave dashboard
to avoid unexpected charges.

See [Web tools](/tools/web).

### 5) Web fetch tool (Firecrawl)

`web_fetch` can call **Firecrawl** when an API key is present:

- `FIRECRAWL_API_KEY` or `plugins.entries.firecrawl.config.webFetch.apiKey`

If Firecrawl isn't configured, the tool falls back to direct fetch plus the bundled `web-readability` plugin (no paid API). Disable `plugins.entries.web-readability.enabled` to skip local Readability extraction.

See [Web tools](/tools/web).

### 6) Provider usage snapshots (status/health)

Some status commands call **provider usage endpoints** to display quota windows or auth health.
These are typically low-volume calls but still hit provider APIs:

- `openclaw status --usage`
- `openclaw models status --json`

See [Models CLI](/cli/models).

### 7) Compaction safeguard summarization

The compaction safeguard can summarize session history using the **current model**, which
invokes provider APIs when it runs.

See [Session management + compaction](/reference/session-management-compaction).

### 8) Model scan / probe

`openclaw models scan` can probe OpenRouter models and uses `OPENROUTER_API_KEY` when
probing is enabled.

See [Models CLI](/cli/models).

### 9) Talk (speech)

Talk mode can invoke **ElevenLabs** when configured:

- `ELEVENLABS_API_KEY` or `talk.providers.elevenlabs.apiKey`

See [Talk mode](/nodes/talk).

### 10) Skills (third-party APIs)

Skills can store `apiKey` in `skills.entries.<name>.apiKey`. If a skill uses that key for external
APIs, it can incur costs according to the skill's provider.

See [Skills](/tools/skills).

## Related

- [Token use and costs](/reference/token-use)
- [Prompt caching](/reference/prompt-caching)
- [Usage tracking](/concepts/usage-tracking)


## Token Use

OpenClaw tracks **tokens**, not characters. Tokens are model-specific, but most
OpenAI-style models average ~4 characters per token for English text.

## How the system prompt is built

OpenClaw assembles its own system prompt on every run. It includes:

- Tool list + short descriptions
- Skills list (only metadata; instructions are loaded on demand with `read`).
  The compact skills block is bounded by `skills.limits.maxSkillsPromptChars`,
  with optional per-agent override at
  `agents.list[].skillsLimits.maxSkillsPromptChars`.
- Self-update instructions
- Workspace + bootstrap files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` when new, plus `MEMORY.md` when present). Lowercase root `memory.md` is not injected; it is legacy repair input for `openclaw doctor --fix` when paired with `MEMORY.md`. Large files are truncated by `agents.defaults.bootstrapMaxChars` (default: 12000), and total bootstrap injection is capped by `agents.defaults.bootstrapTotalMaxChars` (default: 60000). `memory/*.md` daily files are not part of the normal bootstrap prompt; they remain on-demand via memory tools on ordinary turns, but reset/startup model runs can prepend a one-shot startup-context block with recent daily memory for that first turn. Bare chat `/new` and `/reset` commands are acknowledged without invoking the model. The startup prelude is controlled by `agents.defaults.startupContext`.
- Time (UTC + user timezone)
- Reply tags + heartbeat behavior
- Runtime metadata (host/OS/model/thinking)

See the full breakdown in [System Prompt](/concepts/system-prompt).

## What counts in the context window

Everything the model receives counts toward the context limit:

- System prompt (all sections listed above)
- Conversation history (user + assistant messages)
- Tool calls and tool results
- Attachments/transcripts (images, audio, files)
- Compaction summaries and pruning artifacts
- Provider wrappers or safety headers (not visible, but still counted)

Some runtime-heavy surfaces have their own explicit caps:

- `agents.defaults.contextLimits.memoryGetMaxChars`
- `agents.defaults.contextLimits.memoryGetDefaultLines`
- `agents.defaults.contextLimits.toolResultMaxChars`
- `agents.defaults.contextLimits.postCompactionMaxChars`

Per-agent overrides live under `agents.list[].contextLimits`. These knobs are
for bounded runtime excerpts and injected runtime-owned blocks. They are
separate from bootstrap limits, startup-context limits, and skills prompt
limits.

For images, OpenClaw downscales transcript/tool image payloads before provider calls.
Use `agents.defaults.imageMaxDimensionPx` (default: `1200`) to tune this:

- Lower values usually reduce vision-token usage and payload size.
- Higher values preserve more visual detail for OCR/UI-heavy screenshots.

For a practical breakdown (per injected file, tools, skills, and system prompt size), use `/context list` or `/context detail`. See [Context](/concepts/context).

## How to see current token usage

Use these in chat:

- `/status` → **emoji-rich status card** with the session model, context usage,
  last response input/output tokens, and **estimated cost** (API key only).
- `/usage off|tokens|full` → appends a **per-response usage footer** to every reply.
  - Persists per session (stored as `responseUsage`).
  - OAuth auth **hides cost** (tokens only).
- `/usage cost` → shows a local cost summary from OpenClaw session logs.

Other surfaces:

- **TUI/Web TUI:** `/status` + `/usage` are supported.
- **CLI:** `openclaw status --usage` and `openclaw channels list` show
  normalized provider quota windows (`X% left`, not per-response costs).
  Current usage-window providers: Anthropic, GitHub Copilot, Gemini CLI,
  OpenAI Codex, MiniMax, Xiaomi, and z.ai.

Usage surfaces normalize common provider-native field aliases before display.
For OpenAI-family Responses traffic, that includes both `input_tokens` /
`output_tokens` and `prompt_tokens` / `completion_tokens`, so transport-specific
field names do not change `/status`, `/usage`, or session summaries.
Gemini CLI JSON usage is normalized too: reply text comes from `response`, and
`stats.cached` maps to `cacheRead` with `stats.input_tokens - stats.cached`
used when the CLI omits an explicit `stats.input` field.
For native OpenAI-family Responses traffic, WebSocket/SSE usage aliases are
normalized the same way, and totals fall back to normalized input + output when
`total_tokens` is missing or `0`.
When the current session snapshot is sparse, `/status` and `session_status` can
also recover token/cache counters and the active runtime model label from the
most recent transcript usage log. Existing nonzero live values still take
precedence over transcript fallback values, and larger prompt-oriented
transcript totals can win when stored totals are missing or smaller.
Usage auth for provider quota windows comes from provider-specific hooks when
available; otherwise OpenClaw falls back to matching OAuth/API-key credentials
from auth profiles, env, or config.
Assistant transcript entries persist the same normalized usage shape, including
`usage.cost` when the active model has pricing configured and the provider
returns usage metadata. This gives `/usage cost` and transcript-backed session
status a stable source even after the live runtime state is gone.

OpenClaw keeps provider usage accounting separate from the current context
snapshot. Provider `usage.total` can include cached input, output, and multiple
tool-loop model calls, so it is useful for cost and telemetry but can overstate
the live context window. Context displays and diagnostics use the latest prompt
snapshot (`promptTokens`, or the last model call when no prompt snapshot is
available) for `context.used`.

## Cost estimation (when shown)

Costs are estimated from your model pricing config:

```
models.providers.<provider>.models[].cost
```

These are **USD per 1M tokens** for `input`, `output`, `cacheRead`, and
`cacheWrite`. If pricing is missing, OpenClaw shows tokens only. OAuth tokens
never show dollar cost.

After sidecars and channels reach the Gateway ready path, OpenClaw starts an
optional background pricing bootstrap for configured model refs that do not
already have local pricing. That bootstrap fetches remote OpenRouter and LiteLLM
pricing catalogs. Set `models.pricing.enabled: false` to skip those catalog
fetches on offline or restricted networks; explicit
`models.providers.*.models[].cost` entries continue to drive local cost
estimates.

## Cache TTL and pruning impact

Provider prompt caching only applies within the cache TTL window. OpenClaw can
optionally run **cache-ttl pruning**: it prunes the session once the cache TTL
has expired, then resets the cache window so subsequent requests can re-use the
freshly cached context instead of re-caching the full history. This keeps cache
write costs lower when a session goes idle past the TTL.

Configure it in [Gateway configuration](/gateway/configuration) and see the
behavior details in [Session pruning](/concepts/session-pruning).

Heartbeat can keep the cache **warm** across idle gaps. If your model cache TTL
is `1h`, setting the heartbeat interval just under that (e.g., `55m`) can avoid
re-caching the full prompt, reducing cache write costs.

In multi-agent setups, you can keep one shared model config and tune cache behavior
per agent with `agents.list[].params.cacheRetention`.

For a full knob-by-knob guide, see [Prompt Caching](/reference/prompt-caching).

For Anthropic API pricing, cache reads are significantly cheaper than input
tokens, while cache writes are billed at a higher multiplier. See Anthropic's
prompt caching pricing for the latest rates and TTL multipliers:
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Example: keep 1h cache warm with heartbeat

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Example: mixed traffic with per-agent cache strategy

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # default baseline for most agents
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # keep long cache warm for deep sessions
    - id: "alerts"
      params:
        cacheRetention: "none" # avoid cache writes for bursty notifications
```

`agents.list[].params` merges on top of the selected model's `params`, so you can
override only `cacheRetention` and inherit other model defaults unchanged.

### Example: enable Anthropic 1M context beta header

Anthropic's 1M context window is currently beta-gated. OpenClaw can inject the
required `anthropic-beta` value when you enable `context1m` on supported Opus
or Sonnet models.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          context1m: true
```

This maps to Anthropic's `context-1m-2025-08-07` beta header.

This only applies when `context1m: true` is set on that model entry.

Requirement: the credential must be eligible for long-context usage. If not,
Anthropic responds with a provider-side rate limit error for that request.

If you authenticate Anthropic with OAuth/subscription tokens (`sk-ant-oat-*`),
OpenClaw skips the `context-1m-*` beta header because Anthropic currently
rejects that combination with HTTP 401.

## Tips for reducing token pressure

- Use `/compact` to summarize long sessions.
- Trim large tool outputs in your workflows.
- Lower `agents.defaults.imageMaxDimensionPx` for screenshot-heavy sessions.
- Keep skill descriptions short (skill list is injected into the prompt).
- Prefer smaller models for verbose, exploratory work.

See [Skills](/tools/skills) for the exact skill list overhead formula.

## Related

- [API usage and costs](/reference/api-usage-costs)
- [Prompt caching](/reference/prompt-caching)
- [Usage tracking](/concepts/usage-tracking)


## Credits

## The name

OpenClaw = CLAW + TARDIS, because every space lobster needs a time and space machine.

## Credits

- **Peter Steinberger** ([@steipete](https://x.com/steipete)) - Creator, lobster whisperer
- **Mario Zechner** ([@badlogicc](https://x.com/badlogicgames)) - Pi creator, security pen tester
- **Clawd** - The space lobster who demanded a better name

## Core contributors

- **Maxim Vovshin** (@Hyaxia, [36747317+Hyaxia@users.noreply.github.com](mailto:36747317+Hyaxia@users.noreply.github.com)) - Blogwatcher skill
- **Nacho Iacovino** (@nachoiacovino, [nacho.iacovino@gmail.com](mailto:nacho.iacovino@gmail.com)) - Location parsing (Telegram and WhatsApp)
- **Vincent Koc** ([@vincentkoc](https://github.com/vincentkoc), [@vincent_koc](https://x.com/vincent_koc)) - Agents, Telemetry, Hooks, Security

## License

MIT - Free as a lobster in the ocean.

> "We are all just playing with our own prompts." (An AI, probably high on tokens)

## Related

- [Token use and costs](/reference/token-use)
- [Release policy](/reference/RELEASING)

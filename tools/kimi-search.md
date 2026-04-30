---
domain: tools
topic: "Kimi Web Search: AI-Synthesized Answers via Moonshot Web Search"
type: integration
keywords:
  - kimi search
  - moonshot search
  - KIMI_API_KEY
  - MOONSHOT_API_KEY
  - kimi web search provider
  - AI synthesized search
  - citations search
  - kimi-k2.6
source: tools/kimi-search.md
related:
  - tools/web-search-tools
  - providers/moonshot
---

OpenClaw supports Kimi as a `web_search` provider using Moonshot web search to produce AI-synthesized answers with inline citations. Configure with a `KIMI_API_KEY` or `MOONSHOT_API_KEY`.

## Setup

1. Get an API key from [Moonshot AI](https://platform.moonshot.cn/)
2. Set `KIMI_API_KEY` or `MOONSHOT_API_KEY` in the Gateway environment, or run `openclaw configure --section web`

When you choose Kimi during `openclaw onboard` or `openclaw configure --section web`, OpenClaw prompts for:
- Moonshot API region (`https://api.moonshot.ai/v1` or `https://api.moonshot.cn/v1`)
- Default Kimi web-search model (defaults to `kimi-k2.6`)

## Config

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...",          // optional if KIMI_API_KEY or MOONSHOT_API_KEY is set
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## Region Handling

If you use the China API host for chat (`models.providers.moonshot.baseUrl: "https://api.moonshot.cn/v1"`), OpenClaw reuses that same host for Kimi `web_search` when `tools.web.search.kimi.baseUrl` is omitted. This prevents keys from `platform.moonshot.cn` hitting the international endpoint (which returns HTTP 401). Override with `tools.web.search.kimi.baseUrl` when needed.

**Environment alternative**: set `KIMI_API_KEY` or `MOONSHOT_API_KEY` in the Gateway environment (or `~/.openclaw/.env` for daemon installs).

## Supported Parameters

- `query`: required search query
- `count`: accepted for `web_search` compatibility but Kimi returns one synthesized answer with citations regardless of count

## How It Works

Kimi uses Moonshot web search to synthesize answers with inline citations — similar to Gemini and Grok's grounded response approach. Results are a single AI-generated answer rather than a list of N search results.

## Related

- [Web Search overview](/tools/web-search-tools)
- [Moonshot provider](/providers/moonshot)
- [Gemini/Grok search](/tools/web-search-additional)

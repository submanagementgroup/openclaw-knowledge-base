---
domain: tools
topic: "DuckDuckGo Web Search: Key-Free Fallback Search Provider (Experimental)"
type: integration
keywords:
  - duckduckgo
  - duckduckgo search
  - key-free search
  - web_search provider
  - no API key search
  - zero-config search
  - search fallback
  - SafeSearch
source: tools/duckduckgo-search.md
related:
  - tools/web-search-tools
  - tools/brave-search-setup
---

OpenClaw supports DuckDuckGo as a key-free `web_search` provider requiring no API key or account. It is experimental — results are gathered from DuckDuckGo's non-JavaScript HTML search pages rather than an official API.

**Warning**: DuckDuckGo is an unofficial integration. Expect occasional breakage from bot-challenge pages or HTML structure changes.

## Setup

No API key needed. Set DuckDuckGo as provider via `openclaw configure --section web` and select "duckduckgo", or set config directly:

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

## Optional Plugin Config

Configure region and SafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en",       // DuckDuckGo region code
            safeSearch: "moderate", // "strict", "moderate", or "off"
          },
        },
      },
    },
  },
}
```

## Tool Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `query` | required | Search query string |
| `count` | 5 | Results to return (1–10) |
| `region` | — | DuckDuckGo region code (e.g. `us-en`, `uk-en`, `de-de`). Overrides config. |
| `safeSearch` | `moderate` | `strict`, `moderate`, or `off`. Overrides config. |

## Auto-Detection Order

DuckDuckGo is the first key-free fallback (order 100) in auto-detection. API-backed providers with configured keys run first, then Ollama Web Search (order 110), then SearXNG (order 200).

## Notes

- **No API key** — zero configuration required
- **Experimental** — not an official API; results depend on HTML page structure
- **Bot-challenge risk** — DuckDuckGo may serve CAPTCHAs under heavy or automated use
- For production, consider Brave Search (free tier) or another API-backed provider

## Related

- [Web Search overview](/tools/web-search-tools)
- [Brave Search](/tools/brave-search-setup)
- [Tavily](/tools/tavily)

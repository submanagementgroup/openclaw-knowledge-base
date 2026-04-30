---
domain: tools
topic: "Firecrawl: Search, Scrape, and web_fetch Fallback with Bot Circumvention"
type: integration
keywords:
  - firecrawl
  - firecrawl search
  - firecrawl scrape
  - firecrawl_search
  - firecrawl_scrape
  - FIRECRAWL_API_KEY
  - web_fetch fallback
  - anti-bot extraction
  - JS-heavy pages
  - stealth proxy
  - web_search provider
source: tools/firecrawl.md
related:
  - tools/web-search-tools
  - tools/tavily
---

OpenClaw uses Firecrawl in three ways: as the `web_search` provider, as explicit plugin tools `firecrawl_search` and `firecrawl_scrape`, and as a fallback extractor for `web_fetch`. Firecrawl supports bot circumvention and caching for JS-heavy or bot-protected sites.

## Setup

1. Create a Firecrawl account and generate an API key.
2. Store it in config or set `FIRECRAWL_API_KEY` in the gateway environment.

## Configure Firecrawl as web_search Provider

```json5
{
  tools: {
    web: {
      search: {
        provider: "firecrawl",
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
          },
        },
      },
    },
  },
}
```

`web_search` with Firecrawl supports `query` and `count`.

## Configure Firecrawl Scrape and web_fetch Fallback

```json5
{
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000,   // 2 days cache max age
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`firecrawl_scrape` reuses the same `plugins.entries.firecrawl.config.webFetch.*` settings.

## `firecrawl_search` Tool

Use for Firecrawl-specific search controls (core parameters: `query`, `count`, `sources`, `categories`, `scrapeResults`, `timeoutSeconds`).

## `firecrawl_scrape` Tool

Use for JS-heavy or bot-protected pages where plain `web_fetch` is weak (core parameters: `url`, `extractMode`, `maxChars`, `onlyMainContent`, `maxAgeMs`, `proxy`, `storeInCache`, `timeoutSeconds`).

## Stealth / Bot Circumvention

Firecrawl exposes a `proxy` parameter for bot circumvention (`basic`, `stealth`, or `auto`). OpenClaw always uses `proxy: "auto"` plus `storeInCache: true`. `auto` retries with stealth proxies if a basic attempt fails, which may use more credits than basic-only scraping.

## How web_fetch Uses Firecrawl

`web_fetch` extraction order:

1. Readability (local)
2. Firecrawl (if selected or auto-detected as the active web-fetch fallback)
3. Basic HTML cleanup (last fallback)

The selection knob is `tools.web.fetch.provider`. Firecrawl fallback only runs when an API key is available. Legacy `tools.web.fetch.firecrawl.*` config is auto-migrated by `openclaw doctor --fix`.

## Notes

- `FIRECRAWL_BASE_URL` is the shared env fallback for Firecrawl search and scrape base URLs.
- `baseUrl` overrides must stay on `https://api.firecrawl.dev`.

## Related

- [Web Search overview](/tools/web-search-tools)
- [Tavily](/tools/tavily)

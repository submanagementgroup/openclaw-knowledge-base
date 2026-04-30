---
domain: tools
topic: "Tavily: Search and Extract Tools for AI Applications"
type: integration
keywords:
  - tavily
  - tavily search
  - tavily extract
  - tavily_search
  - tavily_extract
  - TAVILY_API_KEY
  - search depth
  - content extraction
  - web_search provider
source: tools/tavily.md
related:
  - tools/web-search-tools
  - tools/firecrawl
---

OpenClaw uses Tavily in two ways: as the `web_search` provider, or as explicit plugin tools `tavily_search` and `tavily_extract`. Tavily is a search API designed for AI applications with configurable depth, topic filtering, domain filters, AI answer summaries, and URL content extraction.

## Setup

1. Create a Tavily account at [tavily.com](https://tavily.com/) and generate an API key.
2. Store it in config or set `TAVILY_API_KEY` in the gateway environment.

## Configure Tavily as web_search Provider

```json5
{
  plugins: {
    entries: {
      tavily: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "tvly-...", // optional if TAVILY_API_KEY is set
            baseUrl: "https://api.tavily.com",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "tavily",
      },
    },
  },
}
```

`web_search` with Tavily supports `query` and `count` (up to 20 results).

## `tavily_search` Tool

Use for Tavily-specific controls instead of generic `web_search`.

| Parameter | Description |
|-----------|-------------|
| `query` | Search query string (keep under 400 characters) |
| `search_depth` | `basic` (default, balanced) or `advanced` (highest relevance, slower) |
| `topic` | `general` (default), `news` (real-time), or `finance` |
| `max_results` | Number of results, 1–20 (default: 5) |
| `include_answer` | Include AI-generated answer summary (default: false) |
| `time_range` | Filter by recency: `day`, `week`, `month`, or `year` |
| `include_domains` | Array of domains to restrict results to |
| `exclude_domains` | Array of domains to exclude |

**Search depth:**

| Depth | Speed | Best for |
|-------|-------|----------|
| `basic` | Faster | General-purpose queries (default) |
| `advanced` | Slower | Precision, specific facts, research |

## `tavily_extract` Tool

Extract clean content from one or more URLs, including JavaScript-rendered pages.

| Parameter | Description |
|-----------|-------------|
| `urls` | Array of URLs to extract (1–20 per request) |
| `query` | Rerank extracted chunks by relevance to this query |
| `extract_depth` | `basic` (default, fast) or `advanced` (for JS-heavy pages) |
| `chunks_per_source` | Chunks per URL, 1–5 (requires `query`) |
| `include_images` | Include image URLs in results (default: false) |

**Extract tips:**
- Max 20 URLs per request. Batch larger lists into multiple calls.
- Use `query` + `chunks_per_source` to get only relevant content instead of full pages.
- Try `basic` first; fall back to `advanced` if content is missing.

## Choosing the Right Tool

| Need | Tool |
|------|------|
| Quick web search, no special options | `web_search` |
| Search with depth, topic, AI answers | `tavily_search` |
| Extract content from specific URLs | `tavily_extract` |

## Related

- [Web Search overview](/tools/web-search-tools)
- [Firecrawl](/tools/firecrawl)

---
domain: tools
topic: "Web Search Tools: Perplexity, Brave, Exa, DuckDuckGo, Tavily, Firecrawl, and More"
type: reference
keywords:
  - web_search
  - Perplexity search
  - Brave search
  - Exa search
  - DuckDuckGo
  - Tavily
  - search tool
related:
  - tools/tools-overview
  - gateway/config-tools-reference
  - providers/perplexity-provider
source:
  - tools/web.md
  - tools/perplexity-search.md
  - tools/brave-search.md
  - tools/exa-search.md
---

Web search tools for OpenClaw agents. Configure the provider via `tools.webSearch.provider` or per-tool API key settings.

## Overview

The `web_search` tool searches the web using your configured provider and
returns results. Results are cached by query for 15 minutes (configurable).

OpenClaw also includes `x_search` for X (formerly Twitter) posts and
`web_fetch` for lightweight URL fetching. In this phase, `web_fetch` stays
local while `web_search` and `x_search` can use xAI Responses under the hood.

  `web_search` is a lightweight HTTP tool, not browser automation. For
  JS-heavy sites or logins, use the [Web Browser](/tools/browser). For
  fetching a specific URL, use [Web Fetch](/tools/web-fetch).

## Quick start

    Pick a provider and complete any required setup. Some providers are
    key-free, while others use API keys. See the provider pages below for
    details.

    ```bash
    openclaw configure --section web
    ```
    This stores the provider and any needed credential. You can also set an env
    var (for example `BRAVE_API_KEY`) and skip this step for API-backed
    providers.

    The agent can now call `web_search`:

    ```javascript
    await web_search({ query: "OpenClaw plugin SDK" });
    ```

    For X posts, use:

    ```javascript
    await x_search({ query: "dinner recipes" });
    ```

## Choosing a provider

    Structured results with snippets. Supports `llm-context` mode, country/language filters. Free tier available.

    Key-free fallback. No API key needed. Unofficial HTML-based integration.

    Neural + keyword search with content extraction (highlights, text, summaries).

    Structured results. Best paired with `firecrawl_search` and `firecrawl_scrape` for deep extraction.

    AI-synthesized answers with citations via Google Search grounding.

    AI-synthesized answers with citations via xAI web grounding.

    AI-synthesized answers with citations via Moonshot web search.

    Structured results via the MiniMax Coding Plan search API.

    Search via a signed-in local Ollama host or the hosted Ollama API.

    Structured results with content extraction c

## Supported Search Providers

## Perplexity Search

# Perplexity Search API

OpenClaw supports Perplexity Search API as a `web_search` provider.
It returns structured results with `title`, `url`, and `snippet` fields.

For compatibility, OpenClaw also supports legacy Perplexity Sonar/OpenRouter setups.
If you use `OPENROUTER_API_KEY`, an `sk-or-...` key in `plugins.entries.perplexity.config.webSearch.apiKey`, or set `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, the provider switches to the chat-completions path and returns AI-synthesized answers with citations instead of structured Search API results.

## Getting a Perplexity API key

1. Create a Perplexity account at [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)
2. Generate an API key in the dashboard
3. Store the key in config or set `PERPLEXITY_API_KEY` in the Gateway environment.

## OpenRouter compatibility

If you were already using OpenRouter for Perplexity Sonar, keep `provider: "perplexity"` and set `OPENROUTER_API_KEY` in the Gateway environment, or store an `sk-or-...` key in `plugins.entries.perplexity.config.webSearch.apiKey`.

Optional compatibility controls:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## Config examples

### Native Perplexity Search API

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
    
## Brave Search

# Brave Search API

OpenClaw supports Brave Search API as a `web_search` provider.

## Get an API key

1. Create a Brave Search API account at [https://brave.com/search/api/](https://brave.com/search/api/)
2. In the dashboard, choose the **Search** plan and generate an API key.
3. Store the key in config or set `BRAVE_API_KEY` in the Gateway environment.

## Config example

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // or "llm-context"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Provider-specific Brave search settings now live under `plugins.entries.brave.config.webSearch.*`.
Legacy `tools.web.search.apiKey` still loads through the compatibility shim, but it is no longer the canonical config path.

`webSearch.mode` controls the Brave transport:

- `web` (default): normal Brave web search with titles, URLs, and snippets
- `llm-context`: Brave LLM Context API with pre-extracted text chunks and sources for grounding

## Tool parameters

<ParamField path="query" type="string" required>
Search query.
</ParamField>

<ParamField path="count" type="number" default="5">
Number of results to return (1–10).
</ParamField>

<ParamField path="country" type="string">
2-letter ISO country code (e.g. `US`, `DE`).
</ParamField>

<Para
## Exa Search

OpenClaw supports [Exa AI](https://exa.ai/) as a `web_search` provider. Exa
offers neural, keyword, and hybrid search modes with built-in content
extraction (highlights, text, summaries).

## Get an API key

    Sign up at [exa.ai](https://exa.ai/) and generate an API key from your
    dashboard.

    Set `EXA_API_KEY` in the Gateway environment, or configure via:

    ```bash
    openclaw configure --section web
    ```

## Config

```json5
{
  plugins: {
    entries: {
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // optional if EXA_API_KEY is set
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**Environment alternative:** set `EXA_API_KEY` in the Gateway environment.
For a gateway install, put it in `~/.openclaw/.env`.

## Tool parameters

<ParamField path="query" type="string" required>
Search query.
</ParamField>

<ParamField path="count" type="number">
Results to return (1–100).
</ParamField>

<ParamField path="type" type="'auto' | 'neural' | 'fast' | 'deep' | 'deep-reasoning' | 'instant'">
Search mode.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Time filter.
</ParamField>

<ParamField path="date_after" type="string">
Results after this date (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Results before this date (`YYYY-MM-DD`).
</ParamField>

<ParamField path="contents" type
## DuckDuckGo Search

OpenClaw supports DuckDuckGo as a **key-free** `web_search` provider. No API
key or account is required.

  DuckDuckGo is an **experimental, unofficial** integration that pulls results
  from DuckDuckGo's non-JavaScript search pages — not an official API. Expect
  occasional breakage from bot-challenge pages or HTML changes.

## Setup

No API key needed — just set DuckDuckGo as your provider:

    ```bash
    openclaw configure --section web
    # Select "duckduckgo" as the provider
    ```

## Config

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

Optional plugin-level settings for region and SafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // DuckDuckGo region code
            safeSearch: "moderate", // "strict", "moderate", or "off"
          },
        },
      },
    },
  },
}
```

## Tool parameters

<ParamField path="query" type="string" required>
Search query.
</ParamField>

<ParamField path="count" type="number" default="5">
Results to return (1–10).
</ParamField>

<ParamField path="region" type="string">
DuckDuckGo region code (e.g. `us-en`, `uk-en`, `de-de`).
</ParamField>

<ParamField path="safeSearch" type="'strict' | 'moderate' | 'off'" default="moderate">
SafeSearch level.
</ParamField>

Region and SafeSearch can also be set in plugin config (see above) — tool
parameters override config values per-query.

## 
## Tavily Search

OpenClaw can use **Tavily** in two ways:

- as the `web_search` provider
- as explicit plugin tools: `tavily_search` and `tavily_extract`

Tavily is a search API designed for AI applications, returning structured results
optimized for LLM consumption. It supports configurable search depth, topic
filtering, domain filters, AI-generated answer summaries, and content extraction
from URLs (including JavaScript-rendered pages).

## Get an API key

1. Create a Tavily account at [tavily.com](https://tavily.com/).
2. Generate an API key in the dashboard.
3. Store it in config or set `TAVILY_API_KEY` in the gateway environment.

## Configure Tavily search

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

Notes:

- Choosing Tavily in onboarding or `openclaw configure --section web` enables
  the bundled Tavily plugin automatically.
- Store Tavily config under `plugins.entries.tavily.config.webSearch.*`.
- `web_search` with Tavily supports `query` and `count` (up to 20 results).
- For Tavily-specific controls like `search_depth`, `topic`, `include_answer`,
  or domain filters, use `tavily_search`.

## Tavily plugin tools

### `tavily_search`

Use this when you want Ta
## Firecrawl

OpenClaw can use **Firecrawl** in three ways:

- as the `web_search` provider
- as explicit plugin tools: `firecrawl_search` and `firecrawl_scrape`
- as a fallback extractor for `web_fetch`

It is a hosted extraction/search service that supports bot circumvention and caching,
which helps with JS-heavy sites or pages that block plain HTTP fetches.

## Get an API key

1. Create a Firecrawl account and generate an API key.
2. Store it in config or set `FIRECRAWL_API_KEY` in the gateway environment.

## Configure Firecrawl search

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

Notes:

- Choosing Firecrawl in onboarding or `openclaw configure --section web` enables the bundled Firecrawl plugin automatically.
- `web_search` with Firecrawl supports `query` and `count`.
- For Firecrawl-specific controls like `sources`, `categories`, or result scraping, use `firecrawl_search`.
- `baseUrl` overrides must stay on `https://api.firecrawl.dev`.
- `FIRECRAWL_BASE_URL` is the shared env fallback for Firecrawl search and scrape base URLs.

## Configure Firecrawl scrape + web_fetch fallback

```json5
{
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config:

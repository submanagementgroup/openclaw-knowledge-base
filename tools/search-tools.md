---
domain: tools
topic: "Web Search Tools: Brave, DuckDuckGo, Exa, Perplexity, Tavily, and More"
type: reference
keywords:
  - web search tools
  - brave search
  - duckduckgo
  - exa
  - perplexity search
  - tavily
  - searxng
  - firecrawl
source: 
  - tools/brave-search.md
  - tools/duckduckgo-search.md
  - tools/exa-search.md
  - tools/firecrawl.md
  - tools/gemini-search.md
  - tools/grok-search.md
  - tools/kimi-search.md
  - tools/minimax-search.md
  - tools/ollama-search.md
  - tools/perplexity-search.md
  - tools/searxng-search.md
  - tools/tavily.md
---

Search tools for web information retrieval. Each tool has its own provider, API key, and configuration.

## Brave Search Tool

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
            baseUrl: "https://api.search.brave.com", // optional proxy/base URL override
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

`webSearch.baseUrl` can point Brave requests at a trusted Brave-compatible proxy
or gateway. OpenClaw appends `/res/v1/web/search` or `/res/v1/llm/context` to
the configured base URL and keeps the base URL in the cache key. Public
endpoints must use `https://`; `http://` is accepted only for trusted loopback
or private-network proxy hosts.

## Tool parameters


Search query.



Number of results to return (1–10).



2-letter ISO country code (e.g. `US`, `DE`).



ISO 639-1 language code for search results (e.g. `en`, `de`, `fr`).



Brave search-language code (e.g. `en`, `en-gb`, `zh-hans`).



ISO language code for UI elements.



Time filter — `day` is 24 hours.



Only results published after this date (`YYYY-MM-DD`).



Only results published before this date (`YYYY-MM-DD`).


**Examples:**

```javascript
// Country and language-specific search
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// Recent results (past week)
await web_search({
  query: "AI news",
  freshness: "week",
});

// Date range search
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## Notes

- OpenClaw uses the Brave **Search** plan. If you have a legacy subscription (e.g. the original Free plan with 2,000 queries/month), it remains valid but does not include newer features like LLM Context or higher rate limits.
- Each Brave plan includes **\$5/month in free credit** (renewing). The Search plan costs \$5 per 1,000 requests, so the credit covers 1,000 queries/month. Set your usage limit in the Brave dashboard to avoid unexpected charges. See the [Brave API portal](https://brave.com/search/api/) for current plans.
- The Search plan includes the LLM Context endpoint and AI inference rights. Storing results to train or tune models requires a plan with explicit storage rights. See the Brave [Terms of Service](https://api-dashboard.search.brave.com/terms-of-service).
- `llm-context` mode returns grounded source entries instead of the normal web-search snippet shape.
- `llm-context` mode supports `freshness` and bounded `date_after` + `date_before` ranges. It does not support `ui_lang`; `date_before` without `date_after` is rejected because Brave requires custom freshness ranges to include both start and end dates.
- `ui_lang` must include a region subtag like `en-US`.
- Results are cached for 15 minutes by default (configurable via `cacheTtlMinutes`).
- Custom `webSearch.baseUrl` values are included in Brave cache identity, so
  proxy-specific responses do not collide.
- Enable the `brave.http` diagnostics flag to log Brave request URLs/query params, response status/timing, and search-cache hit/miss/write events while troubleshooting. The flag never logs the API key or response bodies, but search queries can be sensitive.

## Related

- [Web Search overview](/tools/web) -- all providers and auto-detection
- [Perplexity Search](/tools/perplexity-search) -- structured results with domain filtering
- [Exa Search](/tools/exa-search) -- neural search with content extraction


## DuckDuckGo Search Tool

OpenClaw supports DuckDuckGo as a **key-free** `web_search` provider. No API
key or account is required.

> **Note:** DuckDuckGo is an **experimental, unofficial** integration that pulls results
  from DuckDuckGo's non-JavaScript search pages - not an official API. Expect
  occasional breakage from bot-challenge pages or HTML changes.


## Setup

No API key needed - just set DuckDuckGo as your provider:

**Configure**

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


Search query.



Results to return (1-10).



DuckDuckGo region code (e.g. `us-en`, `uk-en`, `de-de`).



SafeSearch level.


Region and SafeSearch can also be set in plugin config (see above) - tool
parameters override config values per-query.

## Notes

- **No API key** - works out of the box, zero configuration
- **Experimental** - gathers results from DuckDuckGo's non-JavaScript HTML
  search pages, not an official API or SDK
- **Bot-challenge risk** - DuckDuckGo may serve CAPTCHAs or block requests
  under heavy or automated use
- **HTML parsing** - results depend on page structure, which can change without
  notice
- **Auto-detection order** - DuckDuckGo is the first key-free fallback
  (order 100) in auto-detection. API-backed providers with configured keys run
  first, then Ollama Web Search (order 110), then SearXNG (order 200)
- **SafeSearch defaults to moderate** when not configured

> **Note:** For production use, consider [Brave Search](/tools/brave-search) (free tier
  available) or another API-backed provider.


## Related

- [Web Search overview](/tools/web) -- all providers and auto-detection
- [Brave Search](/tools/brave-search) -- structured results with free tier
- [Exa Search](/tools/exa-search) -- neural search with content extraction


## Exa Search Tool

OpenClaw supports [Exa AI](https://exa.ai/) as a `web_search` provider. Exa
offers neural, keyword, and hybrid search modes with built-in content
extraction (highlights, text, summaries).

## Get an API key

**Create an account**

Sign up at [exa.ai](https://exa.ai/) and generate an API key from your
    dashboard.
  
**Store the key**

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
            baseUrl: "https://api.exa.ai", // optional; OpenClaw appends /search
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

## Base URL override

Set `plugins.entries.exa.config.webSearch.baseUrl` when Exa search requests
should go through a compatible proxy or alternate Exa endpoint. OpenClaw
normalizes bare hosts by prepending `https://` and appends `/search` unless the
path already ends there. The resolved endpoint is included in the search cache
key, so results from different Exa endpoints are not shared.

## Tool parameters


Search query.



Results to return (1–100).



Search mode.



Time filter.



Results after this date (`YYYY-MM-DD`).



Results before this date (`YYYY-MM-DD`).



Content extraction options (see below).


### Content extraction

Exa can return extracted content alongside search results. Pass a `contents`
object to enable:

```javascript
await web_search({
  query: "transformer architecture explained",
  type: "neural",
  contents: {
    text: true, // full page text
    highlights: { numSentences: 3 }, // key sentences
    summary: true, // AI summary
  },
});
```

| Contents option | Type                                                                  | Description            |
| --------------- | --------------------------------------------------------------------- | ---------------------- |
| `text`          | `boolean \| { maxCharacters }`                                        | Extract full page text |
| `highlights`    | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | Extract key sentences  |
| `summary`       | `boolean \| { query }`                                                | AI-generated summary   |

### Search modes

| Mode             | Description                       |
| ---------------- | --------------------------------- |
| `auto`           | Exa picks the best mode (default) |
| `neural`         | Semantic/meaning-based search     |
| `fast`           | Quick keyword search              |
| `deep`           | Thorough deep search              |
| `deep-reasoning` | Deep search with reasoning        |
| `instant`        | Fastest results                   |

## Notes

- If no `contents` option is provided, Exa defaults to `{ highlights: true }`
  so results include key sentence excerpts
- Results preserve `highlightScores` and `summary` fields from the Exa API
  response when available
- Result descriptions are resolved from highlights first, then summary, then
  full text — whichever is available
- `freshness` and `date_after`/`date_before` cannot be combined — use one
  time-filter mode
- Up to 100 results can be returned per query (subject to Exa search-type
  limits)
- Results are cached for 15 minutes by default (configurable via
  `cacheTtlMinutes`)
- Exa is an official API integration with structured JSON responses

## Related

- [Web Search overview](/tools/web) -- all providers and auto-detection
- [Brave Search](/tools/brave-search) -- structured results with country/language filters
- [Perplexity Search](/tools/perplexity-search) -- structured results with domain filtering


## Firecrawl Web Scraping Tool

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
- `baseUrl` defaults to hosted Firecrawl at `https://api.firecrawl.dev`. Self-hosted overrides are allowed only for private/internal endpoints; HTTP is accepted only for those private targets.
- `FIRECRAWL_BASE_URL` is the shared env fallback for Firecrawl search and scrape base URLs.

## Configure Firecrawl scrape + web_fetch fallback

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
            maxAgeMs: 172800000,
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

Notes:

- Firecrawl fallback attempts run only when an API key is available (`plugins.entries.firecrawl.config.webFetch.apiKey` or `FIRECRAWL_API_KEY`).
- `maxAgeMs` controls how old cached results can be (ms). Default is 2 days.
- Legacy `tools.web.fetch.firecrawl.*` config is auto-migrated by `openclaw doctor --fix`.
- Firecrawl scrape/base URL overrides follow the same hosted/private rule as search: public hosted traffic uses `https://api.firecrawl.dev`; self-hosted overrides must resolve to private/internal endpoints.
- `firecrawl_scrape` rejects obvious private, loopback, metadata, and non-HTTP(S) target URLs before forwarding them to Firecrawl, matching the `web_fetch` target-safety contract for explicit Firecrawl scrape calls.

`firecrawl_scrape` reuses the same `plugins.entries.firecrawl.config.webFetch.*` settings and env vars.

### Self-hosted Firecrawl

Set `plugins.entries.firecrawl.config.webSearch.baseUrl`,
`plugins.entries.firecrawl.config.webFetch.baseUrl`, or `FIRECRAWL_BASE_URL`
when you run Firecrawl yourself. OpenClaw accepts `http://` only for loopback,
private-network, `.local`, `.internal`, or `.localhost` targets. Public custom
hosts are rejected so Firecrawl API keys are not sent to arbitrary endpoints by
accident.

## Firecrawl plugi
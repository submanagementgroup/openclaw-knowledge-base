---
domain: tools
topic: "MiniMax Web Search: Coding Plan Search API Integration"
type: integration
keywords:
  - minimax search
  - minimax coding plan
  - MINIMAX_CODE_PLAN_KEY
  - MINIMAX_CODING_API_KEY
  - minimax web search provider
  - MiniMax CN global region
  - coding plan search
source: tools/minimax-search.md
related:
  - tools/web-search-tools
  - providers/minimax
---

OpenClaw supports MiniMax as a `web_search` provider through the MiniMax Coding Plan search API. It returns structured search results with titles, URLs, snippets, and related queries.

## Setup

1. Get a MiniMax Coding Plan key from [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key)
2. Set `MINIMAX_CODE_PLAN_KEY` in the Gateway environment, or run `openclaw configure --section web`

OpenClaw also accepts `MINIMAX_CODING_API_KEY` as an env alias. `MINIMAX_API_KEY` is still read as a compatibility fallback when it already points at a coding-plan token.

## Config

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...",  // optional if MINIMAX_CODE_PLAN_KEY is set
            region: "global",    // or "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

## Region Selection

MiniMax Search endpoints:

- **Global**: `https://api.minimax.io/v1/coding_plan/search`
- **CN**: `https://api.minimaxi.com/v1/coding_plan/search`

When `plugins.entries.minimax.config.webSearch.region` is unset, OpenClaw resolves region in this order:

1. `tools.web.search.minimax.region` / plugin-owned `webSearch.region`
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

CN onboarding or `MINIMAX_API_HOST=https://api.minimaxi.com/...` automatically keeps MiniMax Search on the CN host. Even when authenticated through the OAuth `minimax-portal` path, web search still registers as provider id `minimax` — the OAuth provider base URL is only used as a region hint.

## Supported Parameters

- `query`: required search query
- `count`: OpenClaw trims the returned result list to the requested count

## Related

- [Web Search overview](/tools/web-search-tools)
- [MiniMax provider](/providers/minimax)

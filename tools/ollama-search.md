---
domain: tools
topic: "Ollama Web Search: Local or Hosted Search via Ollama API"
type: integration
keywords:
  - ollama web search
  - ollama search provider
  - OLLAMA_API_KEY
  - ollama signin
  - key-free web search
  - hosted ollama search
  - ollama.com search
source: tools/ollama-search.md
related:
  - tools/web-search-tools
  - providers/ollama
---

OpenClaw supports Ollama Web Search as a bundled `web_search` provider using Ollama's web-search API. For local Ollama, no API key is required — only a reachable Ollama host and `ollama signin`. For hosted search, set the Ollama base URL to `https://ollama.com` and provide `OLLAMA_API_KEY`.

## Setup

```bash
# 1. Start Ollama
# 2. Sign in
ollama signin

# 3. Choose Ollama Web Search
openclaw configure --section web
# Select "Ollama Web Search"
```

## Config

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Optional Ollama host override:

```json5
{
  plugins: {
    entries: {
      ollama: {
        config: {
          webSearch: {
            baseUrl: "http://ollama-host:11434",
          },
        },
      },
    },
  },
}
```

If you already configure Ollama as a model provider, the web-search provider reuses that host:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

## Hosted Ollama Web Search

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
      },
    },
  },
}
```

## How Host Resolution Works

- Local Ollama daemon hosts use `/api/experimental/web_search` (signs and forwards to Ollama Cloud).
- `https://ollama.com` hosts use `/api/web_search` directly with bearer API-key auth.
- If no explicit base URL is set, OpenClaw uses `http://127.0.0.1:11434`.
- If the Ollama host is auth-protected, OpenClaw reuses `models.providers.ollama.apiKey` for requests.
- If the configured host doesn't expose web search and `OLLAMA_API_KEY` is set, OpenClaw can fall back to `https://ollama.com/api/web_search`.

## Notes

- No web-search-specific API key field needed for local Ollama.
- Runtime auto-detect can fall back to Ollama Web Search when no higher-priority credentialed provider is configured (order 110, after DuckDuckGo at order 100).
- The Ollama model provider uses `baseUrl` as the canonical key; `baseURL` (no underscore) is also honored for compatibility.

## Related

- [Web Search overview](/tools/web-search-tools)
- [Ollama provider](/providers/ollama)

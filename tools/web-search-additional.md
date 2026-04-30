---
domain: tools
topic: "Additional Search Providers: Gemini, Grok, Kimi, Ollama, SearXNG, MiniMax, and Web Fetch"
type: reference
keywords:
  - Gemini search
  - Grok search
  - SearXNG
  - ollama search
  - web fetch
  - web_fetch tool
related:
  - tools/web-search-tools
source:
  - tools/gemini-search.md
  - tools/grok-search.md
  - tools/searxng-search.md
  - tools/web-fetch.md
---

Additional web search providers for OpenClaw.

## Gemini Search

OpenClaw supports Gemini models with built-in
[Google Search grounding](https://ai.google.dev/gemini-api/docs/grounding),
which returns AI-synthesized answers backed by live Google Search results with
citations.

## Get an API key

    Go to [Google AI Studio](https://aistudio.google.com/apikey) and create an
    API key.

    Set `GEMINI_API_KEY` in the Gateway environment, or configure via:

    ```bash
    openclaw configure --section web
    ```

## Config

```json5
{
  plugins: {
    entries: {
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // optional if GEMINI_API_KEY is set
            model: "gemini-2.5-flash", // default
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**Environment alternative:** set `GEMINI_API_KEY` in the Gateway environment.
For a gateway install, put it in `~/.openclaw/.env`.

## How it works

Unlike traditional search providers that return a list of links and snippets,
Gemini uses Google Search grounding to produce AI-synthesized answers with
inline citations. The results include both the synthesized answer and the source
URLs.

- Citation URLs from Gemini grounding are automatically resolved from Google
  redirect URLs to direct URLs.
- Redirect resolution uses the SSRF guard path (HEAD + redirect checks +
  http/https validation) before returning the final citation URL.
- Redirect resolution uses strict SSRF defa
## Grok Search

OpenClaw supports Grok as a `web_search` provider, using xAI web-grounded
responses to produce AI-synthesized answers backed by live search results
with citations.

The same `XAI_API_KEY` can also power the built-in `x_search` tool for X
(formerly Twitter) post search. If you store the key under
`plugins.entries.xai.config.webSearch.apiKey`, OpenClaw now reuses it as a
fallback for the bundled xAI model provider too.

For post-level X metrics such as reposts, replies, bookmarks, or views, prefer
`x_search` with the exact post URL or status ID instead of a broad search
query.

## Onboarding and configure

If you choose **Grok** during:

- `openclaw onboard`
- `openclaw configure --section web`

OpenClaw can show a separate follow-up step to enable `x_search` with the same
`XAI_API_KEY`. That follow-up:

- only appears after you choose Grok for `web_search`
- is not a separate top-level web-search provider choice
- can optionally set the `x_search` model during the same flow

If you skip it, you can enable or change `x_search` later in config.

## Get an API key

    Get an API key from [xAI](https://console.x.ai/).

    Set `XAI_API_KEY` in the Gateway environment, or configure via:

    ```bash
    openclaw configure --section web
    ```

## Config

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // optional if XAI_API_KEY is set
          },
        },
      },
    },
  },
  tools: {
    web: {
    
## Kimi Search

OpenClaw supports Kimi as a `web_search` provider, using Moonshot web search
to produce AI-synthesized answers with citations.

## Get an API key

    Get an API key from [Moonshot AI](https://platform.moonshot.cn/).

    Set `KIMI_API_KEY` or `MOONSHOT_API_KEY` in the Gateway environment, or
    configure via:

    ```bash
    openclaw configure --section web
    ```

When you choose **Kimi** during `openclaw onboard` or
`openclaw configure --section web`, OpenClaw can also ask for:

- the Moonshot API region:
  - `https://api.moonshot.ai/v1`
  - `https://api.moonshot.cn/v1`
- the default Kimi web-search model (defaults to `kimi-k2.6`)

## Config

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // optional if KIMI_API_KEY or MOONSHOT_API_KEY is set
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

If you use the China API host for chat (`models.providers.moonshot.baseUrl`:
`https://api.moonshot.cn/v1`), OpenClaw reuses that same host for Kimi
`web_search` when `tools.web.search.kimi.baseUrl` is omitted, so keys from
[platform.moonshot.cn](https://platform.moonshot.cn/) do not hit the
international endpoint by mistake (which often returns HTTP 401). Override
with `tools.web.search.kimi.baseUrl` when you need a different search bas
## Ollama Search

OpenClaw supports **Ollama Web Search** as a bundled `web_search` provider. It
uses Ollama's web-search API and returns structured results with titles, URLs,
and snippets.

For local or self-hosted Ollama, this setup does not need an API key by
default. It does require:

- an Ollama host that is reachable from OpenClaw
- `ollama signin`

For direct hosted search, set the Ollama provider base URL to `https://ollama.com`
and provide a real `OLLAMA_API_KEY`.

## Setup

    Make sure Ollama is installed and running.

    Run:

    ```bash
    ollama signin
    ```

    Run:

    ```bash
    openclaw configure --section web
    ```

    Then select **Ollama Web Search** as the provider.

If you already use Ollama for models, Ollama Web Search reuses the same
configured host.

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

If you already configure Ollama as a model provider, the web-search provider can
reuse that host instead:

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

The Ollama model provider uses `baseUrl` as the canonical key. The web-search provider also honors `baseURL` on `models.providers.ollama`
## SearXNG Search

OpenClaw supports [SearXNG](https://docs.searxng.org/) as a **self-hosted,
key-free** `web_search` provider. SearXNG is an open-source meta-search engine
that aggregates results from Google, Bing, DuckDuckGo, and other sources.

Advantages:

- **Free and unlimited** -- no API key or commercial subscription required
- **Privacy / air-gap** -- queries never leave your network
- **Works anywhere** -- no region restrictions on commercial search APIs

## Setup

    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    Or use any existing SearXNG deployment you have access to. See the
    [SearXNG documentation](https://docs.searxng.org/) for production setup.

    ```bash
    openclaw configure --section web
    # Select "searxng" as the provider
    ```

    Or set the env var and let auto-detection find it:

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

## Config

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

Plugin-level settings for the SearXNG instance:

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // optional
            language: "en", // optional
          },
        },
      },
    },
  },
}
```

The `baseUrl` field also accepts SecretRef objects.

Transport rules:

- `https://` works for public or private SearXNG hosts
- `http://` is on
## MiniMax Search

OpenClaw supports MiniMax as a `web_search` provider through the MiniMax
Coding Plan search API. It returns structured search results with titles, URLs,
snippets, and related queries.

## Get a Coding Plan key

    Create or copy a MiniMax Coding Plan key from
    [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key).

    Set `MINIMAX_CODE_PLAN_KEY` in the Gateway environment, or configure via:

    ```bash
    openclaw configure --section web
    ```

OpenClaw also accepts `MINIMAX_CODING_API_KEY` as an env alias. `MINIMAX_API_KEY`
is still read as a compatibility fallback when it already points at a coding-plan token.

## Config

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // optional if MINIMAX_CODE_PLAN_KEY is set
            region: "global", // or "cn"
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

**Environment alternative:** set `MINIMAX_CODE_PLAN_KEY` in the Gateway environment.
For a gateway install, put it in `~/.openclaw/.env`.

## Region selection

MiniMax Search uses these endpoints:

- Global: `https://api.minimax.io/v1/coding_plan/search`
- CN: `https://api.minimaxi.com/v1/coding_plan/search`

If `plugins.entries.minimax.config.webSearch.region` is unset, OpenClaw resolves
the region in this order:

1. `tools.web.search.minimax.region` / plugin-o

## Web Fetch Tool

The `web_fetch` tool does a plain HTTP GET and extracts readable content
(HTML to markdown or text). It does **not** execute JavaScript.

For JS-heavy sites or login-protected pages, use the
[Web Browser](/tools/browser) instead.

## Quick start

`web_fetch` is **enabled by default** -- no configuration needed. The agent can
call it immediately:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## Tool parameters

<ParamField path="url" type="string" required>
URL to fetch. `http(s)` only.
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
Output format after main-content extraction.
</ParamField>

<ParamField path="maxChars" type="number">
Truncate output to this many characters.
</ParamField>

## How it works

    Sends an HTTP GET with a Chrome-like User-Agent and `Accept-Language`
    header. Blocks private/internal hostnames and re-checks redirects.

    Runs Readability (main-content extraction) on the HTML response.

    If Readability fails and Firecrawl is configured, retries through the
    Firecrawl API with bot-circumvention mode.

    Results are cached for 15 minutes (configurable) to reduce repeated
    fetches of the same URL.

## Config

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // default: true
        provider: "firecrawl", // optional; omit for auto-detect
        maxChars: 50000, // max output chars
        maxCharsCap: 50000, // hard cap for maxChars param
        maxResponseBytes: 2000000, // max download size before truncation
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true, // use Readability extraction
        userAgent: "Mozilla/5.0 ...", // override User-Agent
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // opt-in for trusted fake-IP proxies using 198.18.0.0/15
          allowIpv6UniqueLocalRange: true, // opt-in for trusted fake-IP proxies using fc00::/7
        },
      },
    },
  },
}
```

## Firecrawl fallback

If Readability extraction fails, `web_fetch` can fall back to
[Firecrawl](/tools/firecrawl) for bot-circumvention and better extraction:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // optional; omit for auto-detect from available credentials
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            apiKey: "fc-...", // optional if FIRECRAWL_API_KEY is set
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 86400000, // cache duration (1 day)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` supports SecretRef objects.
Legacy `tools.web.fetch.firecrawl.*` config is auto-migrated by `openclaw doctor --fix`.

  If Firecrawl is enabled and its SecretRef is unresolved with no
  `FIRECRAWL_API_KEY` env fallback, gateway startup fails fast.

  Firecrawl `baseUrl` overrides are locked down: they must use `https://` and
  the official Firecrawl host (`api.firecrawl.dev`).

Current runtime behavior:

- `tools.web.fetch.provider` selects the fetch fallback provider explicitly.
- If `provider` is omitted, OpenClaw auto-detects the first ready web-fetch
  provider from available credentials. Today the bundled provider is Firecrawl.
- If Readability is disabled, `web_fetch` skips straight to the selected
  provider fallback. If no provider is available, it fails closed.

## Limits and safety

- `maxChars` is clamped to `tools.web.fetch.maxCharsCap`
- Response body is capped at `maxResponseBytes` before parsing; oversized
  responses are truncated with a warning
- Private/internal hostnames are blocked
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` and
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` are narrow opt-ins
  for trusted fake-IP proxy stacks; leave them unset unless your proxy owns
  those synthetic ranges and enforces its own destination policy
- Redirects are checked and limited by `maxRedirects`
- `web_fetch` is best-effort -- some sites need the [Web Browser](/tools/browser)

## Tool profiles

If you use tool profiles or allowlists, add `web_fetch` or `group:web`:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // or: allow: ["group:web"]  (includes web_fetch, web_search, and x_search)
  },
}
```

## Related

- [Web Search](/tools/web) -- search the web with multiple providers
- [Web Browser](/tools/browser) -- full browser automation for JS-heavy sites
- [Firecrawl](/tools/firecrawl) -- Firecrawl search and scrape tools

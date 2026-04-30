---
domain: providers
topic: "Perplexity Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Perplexity
  - perplexity-provider provider
  - AI provider
  - model config
  - API key
  - Perplexity
  - Sonar
  - perplexity search
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/perplexity-provider.md
---

Perplexity provider setup, configuration, and model reference for OpenClaw.

The Perplexity plugin provides web search capabilities through the Perplexity
Search API or Perplexity Sonar via OpenRouter.

This page is the Perplexity **provider** setup. For the Perplexity **tool** (how the agent uses it), see [Perplexity tool](/tools/perplexity-search).

| Property    | Value                                                                  |
| ----------- | ---------------------------------------------------------------------- |
| Type        | Web search provider (not a model provider)                             |
| Auth        | `PERPLEXITY_API_KEY` (direct) or `OPENROUTER_API_KEY` (via OpenRouter) |
| Config path | `plugins.entries.perplexity.config.webSearch.apiKey`                   |

## Getting started

    Run the interactive web-search configuration flow:

    ```bash
    openclaw configure --section web
    ```

    Or set the key directly:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    The agent will automatically use Perplexity for web searches once the key is
    configured. No additional steps are required.

## Search modes

The plugin auto-selects the transport based on API key prefix:

    When your key starts with `pplx-`, OpenClaw uses the native Perplexity Search
    API. This transport returns structured results and supports domain, language,
    and date filters (see filtering options below).

    When your key starts with `sk-or-`, OpenClaw routes through OpenRouter using
    the Perplexity Sonar model. This transport returns AI-synthesized answers with
    citations.

| Key prefix | Transport                    | Features                                         |
| ---------- | ---------------------------- | ------------------------------------------------ |
| `pplx-`    | Native Perplexity Search API | Structured results, domain/language/date filters |
| `sk-or-`   | OpenRouter (Sonar)           | AI-synthesized answers with citations            |

## Native API filtering

Filtering options are only available when using the native Perplexity API
(`pplx-` key). OpenRouter/Sonar searches do not support these parameters.

When using the native Perplexity API, searches support the following filters:

| Filter         | Description                            | Example                             |
| -------------- | -------------------------------------- | ----------------------------------- |
| Country        | 2-letter country code                  | `us`, `de`, `jp`                    |
| Language       | ISO 639-1 language code                | `en`, `fr`, `zh`                    |
| Date range     | Recency window                         | `day`, `week`, `month`, `year`      |
| Domain filters | Allowlist or denylist (max 20 domains) | `example.com`                       |
| Content budget | Token limits per response / per page   | `max_tokens`, `max_tokens_per_page` |

## Advanced configuration

    If the OpenClaw Gateway runs as a daemon (launchd/systemd), make sure
    `PERPLEXITY_API_KEY` is available to that process.

    A key set only in `~/.profile` will not be visible to a launchd/systemd
    daemon unless that environment is explicitly imported. Set the key in
    `~/.openclaw/.env` or via `env.shellEnv` to ensure the gateway process can
    read it.

    If you prefer to route Perplexity searches through OpenRouter, set an
    `OPENROUTER_API_KEY` (prefix `sk-or-`) instead of a native Perplexity key.
    OpenClaw will detect the prefix and switch to the Sonar transport
    automatically.

    The OpenRouter transport is useful if you already have an OpenRouter account
    and want consolidated billing across multiple providers.

## Related

    How the agent invokes Perplexity searches and interprets results.

    Full configuration reference including plugin entries.

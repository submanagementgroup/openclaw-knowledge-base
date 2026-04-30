---
domain: memory
topic: "Memory Search: Hybrid BM25 + Vector Embeddings, Embedding Providers, and memory_search Tool"
type: reference
keywords:
  - memory_search
  - memory search
  - vector search
  - BM25
  - hybrid search
  - embeddings
  - sqlite-vec
  - memorySearch.provider
related:
  - memory/memory-qmd
  - memory/memory-config-reference
  - memory/active-memory
source:
  - concepts/memory-search.md
  - concepts/memory-builtin.md
---

OpenClaw memory search uses hybrid BM25 + vector embeddings to retrieve relevant notes. The builtin engine (default) stores the index in a SQLite database per agent. Configure the embedding provider to enable vector search.

## Quick Start

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",  // enables vector search; auto-detected if key is set
      }
    }
  }
}
```

Supported embedding providers: `openai`, `gemini`, `voyage`, `mistral`, `local`, `ollama`, `deepinfra`.

## Builtin Engine Features

The builtin engine is the default memory backend. It stores your memory index in
a per-agent SQLite database and needs no extra dependencies to get started.

## What it provides

- **Keyword search** via FTS5 full-text indexing (BM25 scoring).
- **Vector search** via embeddings from any supported provider.
- **Hybrid search** that combines both for best results.
- **CJK support** via trigram tokenization for Chinese, Japanese, and Korean.
- **sqlite-vec acceleration** for in-database vector queries (optional).

## Getting started

If you have an API key for OpenAI, Gemini, Voyage, Mistral, or DeepInfra, the builtin
engine auto-detects it and enables vector search. No config needed.

To set a provider explicitly:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
      },
    },
  },
}
```

Without an embedding provider, only keyword search is available.

To force the built-in local embedding provider, install the optional
`node-llama-cpp` runtime package next to OpenClaw, then point `local.modelPath`
at a GGUF file:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "local",
        fallback: "none",
        local: {
          modelPath: "~/.node-llama-cpp/models/embeddinggemma-300m-qat-Q8_0.gguf",
        },
      },
    },
  },
}
```

## Supported embedding providers

| Provider  | ID          | Auto-detected | Notes                               |
| --------- | ----------- | ------------- | ----------------------------------- |
| OpenAI    | `openai`    | Yes           | Default: `text-embedding-3-small`   |
| Gemini    | `gemini`    | Yes           | Supports multimodal (image + audio) |
| Voyage    | `voyage`    | Yes           |                                     |
| Mistral   | `mistral`   | Yes           |                                     |
| DeepInfra | `deepinfra` | Yes           | Default: `BAAI/bge-m3`              |
| Ollama    | `ollama`    | No            | Local, set explicitly               |
| Local     | `local`     | Yes (first)   | Optional `node-llama-cpp` runtime   |

Auto-detection picks the first provider whose API key can be resolved, in the
order shown. Set `memorySearch.provider` to override.

## How indexing works

OpenClaw indexes `MEMORY.md` and `memory/*.md` into chunks (~400 tokens with
80-token overlap) and stores them in a per-agent SQLite database.

- **Index location:** `~/.openclaw/memory/<agentId>.sqlite`
- **Storage maintenance:** SQLite WAL sidecars are bounded with periodic and
  shutdown checkpoints.
- **File watching:** changes to memory files trigger a debounced reindex (1.5s).
- **Auto-reindex:** when the embedding provider, model, or chunking config
  changes, the entire index is rebuilt automatically.
- **Reindex on demand:** `openclaw memory index --force`

You can also index Markdown files outside the workspace with
`memorySearch.extraPaths`. See the
[configuration reference](/reference/memory-config#additional-memory-paths).

## When to use

The builtin engine is the right choice for most users:

- Works out of the box with no extra dependencies.
- Handles keyword and vector search well.
- Supports all embedding providers.
- Hybrid search combines the best of both retrieval approaches.

Consider switching to [QMD](/concepts/memory-qmd) if you need reranking, query
expansion, or want to index directories outside the workspace.

Consider [Honcho](/concepts/memory-honcho) if you want cross-session memory with
automatic user modeling.

## Troubleshooting

**Memory search disabled?** Check `openclaw memory status`. If no provider is
detected, set one explicitly or add an API key.

**Local provider not detected?** Confirm the local path exists and run:

```bash
openclaw memory status --deep --agent main
openclaw memory index --force --agent main
```

Both standalone CLI commands and the Gateway use the same `local` provider id.
If the provider is set to `auto`, local embeddings are considered first only
when `me

## Memory Search Configuration

`memory_search` finds relevant notes from your memory files, even when the
wording differs from the original text. It works by indexing memory into small
chunks and searching them using embeddings, keywords, or both.

## Quick start

If you have a GitHub Copilot subscription, OpenAI, Gemini, Voyage, or Mistral
API key configured, memory search works automatically. To set a provider
explicitly:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai", // or "gemini", "local", "ollama", etc.
      },
    },
  },
}
```

For multi-endpoint setups, `provider` can also be a custom
`models.providers.<id>` entry, such as `ollama-5080`, when that provider sets
`api: "ollama"` or another embedding adapter owner.

For local embeddings with no API key, install the optional `node-llama-cpp`
runtime package next to OpenClaw and use `provider: "local"`.

Some OpenAI-compatible embedding endpoints require asymmetric labels such as
`input_type: "query"` for searches and `input_type: "document"` or `"passage"`
for indexed chunks. Configure those with `memorySearch.queryInputType` and
`memorySearch.documentInputType`; see the [Memory configuration reference](/reference/memory-config#provider-specific-config).

## Supported providers

| Provider       | ID               | Needs API key | Notes                                                |
| -------------- | ---------------- | ------------- | ---------------------------------------------------- |
| Bedrock        | `bedrock`        | No            | Auto-detected when the AWS credential chain resolves |
| Gemini         | `gemini`         | Yes           | Supports image/audio indexing                        |
| GitHub Copilot | `github-copilot` | No            | Auto-detected, uses Copilot subscription             |
| Local          | `local`          | No            | GGUF model, ~0.6 GB download                         |
| Mistral        | `mistral`        | Yes           | Auto-detected                                        |
| Ollama         | `ollama`         | No            | Local, must set explicitly                           |
| OpenAI         | `openai`         | Yes           | Auto-detected, fast                                  |
| Voyage         | `voyage`         | Yes           | Auto-detected                                        |

## How search works

OpenClaw runs two retrieval paths in parallel and merges the results:

- **Vector search** finds notes with similar meaning ("gateway host" matches
  "the machine running OpenClaw").
- **BM25 keyword search** finds exact matches (IDs, error strings, config
  keys).

If only one path is available (no embeddings or no FTS), the other runs alone.

When embeddings are unavailable, OpenClaw still uses lexical ranking over FTS results instead of falling back to raw exact-match ordering only. That degraded mode boosts chunks with stronger query-term coverage and relevant file paths, which keeps recall useful even without `sqlite-vec` or an embedding provider.

## Improving search quality

Two optional features help when you have a large note history:

### Temporal decay

Old notes gradually lose ranking weight so recent information surfaces first.
With the default half-life of 30 days, a note from last month scores at 50% of
its original weight. Evergreen files like `MEMORY.md` are never decayed.

Enable temporal decay if your agent has months of daily notes and stale
information keeps outranking recent context.

### MMR (diversity)

Reduces redundant results. If five notes all mention the same router config, MMR
ensures the top results cover different topics instead of repeating.

Enable MMR if `memory_search` keeps returning near-duplicate snippets from
different daily notes.

### Enable both

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            mmr: { enabled: true },
            temporalDecay: { enabled: true },
          },
        },

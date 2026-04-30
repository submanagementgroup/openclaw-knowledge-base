---
domain: memory
topic: "QMD Memory Engine: Local Search Sidecar with Reranking, Query Expansion, and Extra Index Paths"
type: procedure
keywords:
  - QMD
  - memory QMD
  - reranking
  - query expansion
  - extra paths
  - local search
  - memory backend
related:
  - memory/memory-search
  - plugins/memory-lancedb
source: concepts/memory-qmd.md
---

QMD is a local-first memory search sidecar for OpenClaw that adds reranking, query expansion, and the ability to index extra directories beyond the workspace. It is an alternative to the builtin SQLite engine.

[QMD](https://github.com/tobi/qmd) is a local-first search sidecar that runs
alongside OpenClaw. It combines BM25, vector search, and reranking in a single
binary, and can index content beyond your workspace memory files.

## What it adds over builtin

- **Reranking and query expansion** for better recall.
- **Index extra directories** -- project docs, team notes, anything on disk.
- **Index session transcripts** -- recall earlier conversations.
- **Fully local** -- runs with the optional node-llama-cpp runtime package and
  auto-downloads GGUF models.
- **Automatic fallback** -- if QMD is unavailable, OpenClaw falls back to the
  builtin engine seamlessly.

## Getting started

### Prerequisites

- Install QMD: `npm install -g @tobilu/qmd` or `bun install -g @tobilu/qmd`
- SQLite build that allows extensions (`brew install sqlite` on macOS).
- QMD must be on the gateway's `PATH`.
- macOS and Linux work out of the box. Windows is best supported via WSL2.

### Enable

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw creates a self-contained QMD home under
`~/.openclaw/agents/<agentId>/qmd/` and manages the sidecar lifecycle
automatically -- collections, updates, and embedding runs are handled for you.
It prefers current QMD collection and MCP query shapes, but still falls back to
alternate collection pattern flags and older MCP tool names when needed.
Boot-time reconciliation also recreates stale managed collections back to their
canonical patterns when an older QMD collection with the same name is still
present.

## How the sidecar works

- OpenClaw creates collections from your workspace memory files and any
  configured `memory.qmd.paths`, then runs `qmd update` when the QMD manager is
  opened and periodically afterward (default every 5 minutes). These refreshes
  run through QMD subprocesses, not an in-process filesystem crawl. Semantic
  modes also run `qmd embed`.
- The default workspace collection tracks `MEMORY.md` plus the `memory/`
  tree. Lowercase `memory.md` is not indexed as a root memory file.
- QMD's own scanner ignores hidden paths and common dependency/build
  directories such as `.git`, `.cache`, `node_modules`, `vendor`, `dist`, and
  `build`. Gateway startup does not initialize QMD by default, so cold boot
  avoids importing the memory runtime or creating the long-lived watcher before
  memory is first used.
- If you want a gateway-start refresh anyway, set
  `memory.qmd.update.startup` to `idle` or `immediate`. The opt-in startup
  refresh uses a one-shot QMD subprocess path instead of creating the full
  long-lived in-process watcher.
- Searches use the configured `searchMode` (default: `search`; also supports
  `vsearch` and `query`). `search` is BM25-only, so OpenClaw skips semantic
  vector readiness probes and embedding maintenance in that mode. If a mode
  fails, OpenClaw retries with `qmd query`.
- With QMD releases that advertise multi-collection filters, OpenClaw groups
  same-source collections into one QMD search invocation. Older QMD releases
  keep the compatible per-collection fallback.
- If QMD fails entirely, OpenClaw falls back to the builtin SQLite engine.
  Repeated chat-turn attempts back off briefly after an open failure so a
  missing binary or broken sidecar dependency does not create a retry storm;
  `openclaw memory status` and one-shot CLI probes still recheck QMD directly.

The first search may be slow -- QMD auto-downloads GGUF models (~2 GB) for
reranking and query expansion on the first `qmd query` run.

## Search performance and compatibility

OpenClaw keeps the QMD search path compatible with both current and older QMD
installs.

On startup, OpenClaw checks the installed QMD help text once per manager. If the
binary advertises support for multiple collection filters, OpenClaw searches all
same-source collections with one command:

```bash
qmd search "router notes" --json -n 10 -c memory-root-main -c memory-dir-main
```

This avoids starting one QMD subprocess for every durable-memory collection.
Session transcript collections stay in their own source group, so mixed
`memory` + `sessions` searches still give the result diversifier input from both
sources.

Older QMD builds only accept one collection filter. When OpenClaw detects one
of those builds, it keeps the compatibility path and searches each collection
separately before merging and deduplicating results.

To inspect the installed contract manually, run:

```bash
qmd --help | grep -i collection
```

Current QMD help says collection filters can target one or more collections.
Older help usually describes a single collection.

## Model overrides

QMD model environment variables pass through unchanged from the gateway
process, so you can tune QMD globally without adding new OpenClaw config:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

After changing the embedding model, rerun embeddings so the index matches the
new vector space.

## Indexing extra paths

Point QMD at additional directories to make them searchable:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Snippets from extra paths appear as `qmd/<collection>/<relative-path>` in
search results. `memory_get` understands this prefix and reads from the correct
collection root.

## Indexing session transcripts

Enable session indexing to recall earlier conversations:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Transcripts are exported as sanitized User/Assistant turns into a dedicated QMD
collection under `~/.openclaw/agents/<id>/qmd/sessions/`.

## Search scope

By default, QMD search results are surfaced in direct and channel sessions
(not groups). Configure `memory.qmd.scope` to change this:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

When scope denies a search, OpenClaw logs a warning with the derived channel and
chat type so empty results are easier to debug.

## Citations

When `memory.citations` is `auto` or `on`, search snippets include a
`Source: <path#line>` footer. Set `memory.citations = "off"` to omit the footer
while still passing the path to the agent internally.

## When to use

Choose QMD when you need:

- Reranking for higher-quality results.
- To search project docs or notes outside the workspace.
- To recall past session conversations.
- Fully local search with no API keys.

For simpler setups, the [builtin engine](/concepts/memory-builtin) works well
with no extra dependencies.

## Troubleshooting

**QMD not found?** Ensure the binary is on the gateway's `PATH`. If OpenClaw
runs as a service, create a symlink:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

If `qmd --version` works in your shell but OpenClaw still reports
`spawn qmd ENOENT`, the gateway process likely has a different `PATH` than your
interactive shell. Pin the binary explicitly:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      command: "/absolute/path/to/qmd",
    },
  },
}
```

Use `command -v qmd` in the environment where QMD is installed, then recheck
with `openclaw memory status --deep`.

**First search very slow?** QMD downloads GGUF models on first use. Pre-warm
with `qmd query "test"` using the same XDG dirs OpenClaw uses.

**Many QMD subprocesses during search?** Update QMD if possible. OpenClaw uses
one process for same-source multi-collection searches only when the installed
QMD advertises support for multiple `-c` filters; otherwise it keeps the older
per-collection fallback for correctness.

**BM25-only QMD still trying to build llama.cpp?** Set
`memory.qmd.searchMode = "search"`. OpenClaw treats that mode as

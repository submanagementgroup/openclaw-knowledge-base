# OpenClaw Knowledge Base

Structured knowledge base for the OpenClaw memory search system (`memory_search` / `memory_get`). Built from the OpenClaw documentation source and optimized for hybrid BM25 + vector retrieval.

## Build Info

| Field | Value |
|-------|-------|
| **Last update** | May 1, 2026 (delta v2026.4.27 → v2026.4.29) |
| **Source version** | OpenClaw v2026.4.29 |
| **Source files processed** | 478 `.md` files across 23 source directories |
| **Output KB files** | 339 content files + 20 index files = **359 total** |
| **Domains** | 18 |
| **Source excluded** | `assets/`, `images/`, `snippets/` (binary/empty), `.generated/`, `.i18n/` |

## Domains

| Domain | Files | Description |
|--------|-------|-------------|
| `automation/` | 6 | Cron jobs, heartbeat, hooks, tasks, webhooks, and standing orders |
| `channels/` | 32 | Messaging channels: Telegram, WhatsApp, Discord, Slack, Matrix, Teams, and 20+ more |
| `cli/` | 30 | CLI command reference for all `openclaw` subcommands |
| `concepts/` | 29 | Core concepts: agent loop, memory, sessions, models, context, multi-agent routing |
| `gateway/` | 23 | Gateway service: config, security, secrets, sandboxing, protocols, APIs, CLI backends |
| `getting-started/` | 7 | Quickstart, installation, onboarding wizard, workspace bootstrap, CLI automation |
| `install/` | 14 | Platform-specific install: Docker, Kubernetes, cloud VPS, Nix, package managers |
| `internals/` | 9 | Pi integration, CI pipeline (4 files), QA automation, design plans, release process |
| `memory/` | 7 | Memory system: search, QMD engine, active memory, LanceDB, memory wiki |
| `nodes/` | 6 | iOS, Android, macOS nodes: camera, audio, voice wake, media, location, troubleshooting |
| `platforms/` | 10 | macOS app, iOS, Android, Linux, Windows, Raspberry Pi, DigitalOcean, Oracle Cloud |
| `plugins/` | 32 | Plugin SDK, architecture, building channels/providers/tools, bundled plugins |
| `providers/` | 58 | AI providers: OpenAI, Anthropic, Google, Ollama, Z.AI, and 50+ more |
| `reference/` | 19 | API reference, templates, token usage, prompt caching, SDK design |
| `security/` | 5 | Threat model, security audit, network proxy, formal verification |
| `tools/` | 36 | Agent tools: exec, browser, search, TTS, image/video, skills, search providers |
| `troubleshooting/` | 15 | FAQ, WSL2 browser, first-run issues, model errors, debugging, testing |
| `web-ui/` | 2 | Control UI dashboard, WebChat, and terminal TUI |

## File Format

Every KB file follows this structure:

```markdown
---
domain: <domain-slug>
topic: "<Human-readable topic>"
type: concept | procedure | reference | troubleshooting | integration
keywords:
  - keyword1
  - keyword2
related:
  - domain/other-file
source: path/to/original/source.md
---

[Summary block: 1-3 sentences naming the topic with key terms]

## Specific H2 Header (not "Overview")

Content...
```

## Deployment

### Memory directory (auto-indexed)

```bash
cp -r . ~/.openclaw/workspace/memory/kb/
```

### Extra paths (keeps KB separate from daily memory)

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["/path/to/openclaw-knowledge-base"]
      }
    }
  }
}
```

### QMD collection (advanced — BM25 + vectors + reranking)

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [
        { name: "openclaw-kb", path: "/path/to/openclaw-knowledge-base", pattern: "**/*.md" }
      ]
    }
  }
}
```

## Indexes

- `_index.md` — master manifest with all 347 files listed in a table
- `_domains.md` — one section per domain with file listings
- `<domain>/_index.md` — per-domain file list in each domain folder

## Source Coverage

All 477 source files were processed. Mapping approach:

- **1:1**: Files in the 800–3000 token target range mapped directly (majority of files)
- **Merged**: Small files (<800 tokens) merged with thematically related content
- **Split**: Large files (>12,000 chars) split at H2/H3 boundaries into multiple KB files
- **Excluded**: `tts.md` (redirect-only, 185 bytes), `AGENTS.md`/`CLAUDE.md` (identical meta-docs, merged into one)

Large files split into multiple KB entries:
- `help/faq.md` (84KB) → `troubleshooting/faq.md` + `troubleshooting/faq-2.md`
- `plugins/manifest.md` (76KB) → `plugins/plugin-manifest.md` + `plugins/plugin-manifest-2.md`
- `gateway/security/index.md` (70KB) → `gateway/security-overview.md` + `gateway/security-audit.md`
- `gateway/config-agents.md` (62KB) → `gateway/config-agents-reference.md`
- `plugins/architecture-internals.md` (58KB) → `plugins/architecture-internals.md` + `plugins/architecture-internals-2.md`
- `channels/discord.md` (46KB) → `channels/discord.md` + `channels/discord-advanced.md`
- And 10+ more large files similarly split

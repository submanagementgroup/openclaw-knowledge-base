# OpenClaw Knowledge Base

Structured knowledge base for the OpenClaw memory search system (`memory_search` / `memory_get`). Built from the OpenClaw documentation source and optimized for hybrid BM25 + vector retrieval.

## Build Info

| Field | Value |
|-------|-------|
| **Last update** | May 15, 2026 (full rebuild v2026.5.12) |
| **Source version** | OpenClaw v2026.5.12 (commit 0358f3d) |
| **Source files processed** | 631 `.md` files across 26+ source directories |
| **Output KB files** | 429 content files + 20 index files = **449 total** |
| **Domains** | 18 |
| **Source excluded** | `assets/`, `images/`, `snippets/` (binary/empty), `.generated/`, `.i18n/` |
| **Rebuild type** | Full (clean rebuild from new source, not delta) |

## Domains

| Domain | Files | Description |
|--------|-------|-------------|
| `automation/` | 7 | Cron jobs, heartbeat, hooks, tasks, ClawFlow, webhooks, standing orders |
| `channels/` | 42 | Messaging channels: Telegram, WhatsApp, Discord, Slack, Matrix, Teams, and 25+ more |
| `cli/` | 24 | CLI command reference for all `openclaw` subcommands |
| `concepts/` | 37 | Core concepts: agent loop, memory, sessions, models, context, multi-agent routing |
| `gateway/` | 33 | Gateway service: config, secrets, sandboxing, protocols, OpenAI API, heartbeat, diagnostics, operations |
| `getting-started/` | 16 | Quickstart, installation, onboarding wizard, workspace bootstrap, CLI automation |
| `install/` | 23 | Platform-specific install: Docker, Kubernetes, cloud VPS, Nix, package managers |
| `internals/` | 21 | Design docs, QA automation, SDK API design, release process, refactor docs, specs |
| `memory/` | 5 | Memory system: search, QMD engine, LanceDB, Honcho, memory-core |
| `nodes/` | 6 | Hardware nodes: camera, audio, voice wake, media understanding, location |
| `platforms/` | 9 | macOS app, iOS, Android, Linux, Windows, Raspberry Pi, macOS features |
| `plugins/` | 53 | Plugin SDK (13 files), architecture, manifest, codex harness, 8 batched ref files (117 plugins) |
| `providers/` | 57 | AI providers: OpenAI, Anthropic, Google, Ollama, xAI, Tencent, and 50+ more |
| `reference/` | 19 | Memory config, RPC, rich output, session management, costs, SecretRef, templates, testing |
| `security/` | 12 | Threat model, security model, access controls, network hardening, prompt injection defense, configuration hardening, audit checks, secure file ops, incident response, formal verification |
| `tools/` | 45 | Agent tools: exec, browser, TTS, image/video, skills, search providers, ACP, subagents |
| `troubleshooting/` | 16 | FAQ, first-run issues, model errors, debugging, automation troubleshooting, diagnostics |
| `web-ui/` | 5 | Control UI dashboard, WebChat, TUI, and control UI features |

## What Changed from v2026.4.29

- **plugins/** expanded from 31 → 161 source files (52 KB files, including 8 batched reference files covering 117 plugin stubs)
- **New source directories**: `refactor/` (4 files → `internals/`), `announcements/` (1 file → `reference/`), `clawhub/` (1 file → `tools/`)
- **New KB files**: `concepts/active-memory-config.md`, `tools/acp-agents.md` + `acp-agents-advanced.md`, expanded CLI coverage
- Total source coverage: 631 source files → 429 content KB files (629/632 = 99.5% coverage; 3 redirect-only stubs not referenced)

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

- `_index.md` — master manifest listing all 429 content files
- `_domains.md` — one section per domain with file counts and content types
- `<domain>/_index.md` — per-domain file list in each of the 18 domain folders

## Source Coverage

631 source files processed. Coverage approach:

- **1:1**: Files in the 800–3000 token target range mapped directly (majority of files)
- **Merged**: Small files (<800 tokens) merged with thematically related content
- **Split**: Large files (>12,000 chars) split at H2/H3 boundaries into multiple KB files
- **Batched**: 117 plugin reference stub files (300-700 chars each) batched into 8 grouped KB files
- **Redirect-only stubs kept**: ~11 redirect-only files kept as small stubs for BM25 keyword routing

Notable large files split into multiple KB entries:
- `concepts/active-memory.md` (35KB) → `concepts/active-memory.md` + `concepts/active-memory-config.md`
- `channels/discord.md` (70KB) → `channels/discord.md` + `channels/discord-advanced.md`
- `channels/slack.md` (53KB) → `channels/slack.md` + `channels/slack-advanced.md`
- `channels/telegram.md` (46KB) → `channels/telegram.md` + `channels/telegram-advanced.md`
- `channels/msteams.md` (44KB) → `channels/msteams.md` + `channels/msteams-advanced.md`
- `channels/matrix.md` (44KB) → `channels/matrix.md` + `channels/matrix-advanced.md`
- `plugins/manifest.md` (87KB) → `plugins/manifest.md` + `plugins/manifest-extended.md`
- `gateway/configuration-reference.md` (64KB) → `gateway/configuration-reference.md`
- `tools/tts.md` (45KB) → `tools/tts.md` + `tools/tts-advanced.md`
- `tools/acp-agents.md` (52KB) → `tools/acp-agents.md` + `tools/acp-agents-advanced.md`
- `tools/slash-commands.md` (30KB) → `tools/slash-commands.md` + `tools/slash-commands-2.md`

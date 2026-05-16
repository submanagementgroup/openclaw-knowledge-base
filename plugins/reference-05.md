---
domain: plugins
topic: "Plugin Reference Batch 5: migrate-claude, migrate-hermes, minimax, mistral..."
type: reference
keywords:
  - migrate-claude
  - migrate-hermes
  - minimax
  - mistral
  - moonshot
  - msteams
  - nextcloud-talk
  - nostr
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/migrate-claude.md
  - plugins/reference/migrate-hermes.md
  - plugins/reference/minimax.md
  - plugins/reference/mistral.md
  - plugins/reference/moonshot.md
  - plugins/reference/msteams.md
  - plugins/reference/nextcloud-talk.md
  - plugins/reference/nostr.md
  - plugins/reference/nvidia.md
  - plugins/reference/oc-path.md
  - plugins/reference/ollama.md
  - plugins/reference/open-prose.md
  - plugins/reference/openai.md
  - plugins/reference/opencode-go.md
  - plugins/reference/opencode.md
---

Plugin configuration reference entries for: migrate-claude, migrate-hermes, minimax, mistral, moonshot, msteams, nextcloud-talk, nostr.

## migrate-claude

# Migrate Claude plugin

Imports Claude Code and Claude Desktop instructions, MCP servers, skills, and safe configuration into OpenClaw.

## Distribution

- Package: `@openclaw/migrate-claude`
- Install route: included in OpenClaw

## Surface

contracts: migrationProviders

## migrate-hermes

# Migrate Hermes plugin

Imports Hermes configuration, memories, skills, and supported credentials into OpenClaw.

## Distribution

- Package: `@openclaw/migrate-hermes`
- Install route: included in OpenClaw

## Surface

contracts: migrationProviders

## minimax

# MiniMax plugin

Adds MiniMax, MiniMax Portal model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/minimax-provider`
- Install route: included in OpenClaw

## Surface

providers: minimax, minimax-portal; contracts: imageGenerationProviders, mediaUnderstandingProviders, musicGenerationProviders, speechProviders, videoGenerationProviders, webSearchProviders

## Related docs

- [minimax](/providers/minimax)

## mistral

# Mistral plugin

Adds Mistral model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/mistral-provider`
- Install route: included in OpenClaw

## Surface

providers: mistral; contracts: mediaUnderstandingProviders, memoryEmbeddingProviders, realtimeTranscriptionProviders

## Related docs

- [mistral](/providers/mistral)

## moonshot

# Moonshot plugin

Adds Moonshot model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/moonshot-provider`
- Install route: included in OpenClaw

## Surface

providers: moonshot; contracts: mediaUnderstandingProviders, webSearchProviders

## Related docs

- [moonshot](/providers/moonshot)

## msteams

# Microsoft Teams plugin

Adds the Microsoft Teams channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/msteams`
- Install route: npm; ClawHub

## Surface

channels: msteams

## Related docs

- [msteams](/channels/msteams)

## nextcloud-talk

# Nextcloud Talk plugin

Adds the Nextcloud Talk channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/nextcloud-talk`
- Install route: npm; ClawHub

## Surface

channels: nextcloud-talk

## Related docs

- [nextcloud-talk](/channels/nextcloud-talk)

## nostr

# Nostr plugin

Adds the Nostr channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/nostr`
- Install route: npm; ClawHub

## Surface

channels: nostr

## Related docs

- [nostr](/channels/nostr)

## nvidia

# NVIDIA plugin

Adds NVIDIA model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/nvidia-provider`
- Install route: included in OpenClaw

## Surface

providers: nvidia

## Related docs

- [nvidia](/providers/nvidia)

## oc-path

# Oc Path plugin

Adds the openclaw path CLI for oc:// workspace file addressing.

## Distribution

- Package: `@openclaw/oc-path`
- Install route: included in OpenClaw

## Surface

plugin

## Related docs

- [oc-path](/plugins/oc-path)

## ollama

# Ollama plugin

Adds Ollama model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/ollama-provider`
- Install route: included in OpenClaw

## Surface

providers: ollama; contracts: memoryEmbeddingProviders, webSearchProviders

## Related docs

- [ollama](/providers/ollama)

## open-prose

# Open Prose plugin

OpenProse VM skill pack with a /prose slash command.

## Distribution

- Package: `@openclaw/open-prose`
- Install route: included in OpenClaw

## Surface

skills

## openai

# OpenAI plugin

Adds OpenAI, OpenAI Codex model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/openai-provider`
- Install route: included in OpenClaw

## Surface

providers: openai, openai-codex; contracts: imageGenerationProviders, mediaUnderstandingProviders, memoryEmbeddingProviders, realtimeTranscriptionProviders, realtimeVoiceProviders, speechProviders, videoGenerationProviders

## Related docs

- [openai](/providers/openai)

## opencode-go

# OpenCode Go plugin

Adds OpenCode Go model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/opencode-go-provider`
- Install route: included in OpenClaw

## Surface

providers: opencode-go; contracts: mediaUnderstandingProviders

## Related docs

- [opencode-go](/providers/opencode-go)

## opencode

# OpenCode plugin

Adds OpenCode model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/opencode-provider`
- Install route: included in OpenClaw

## Surface

providers: opencode; contracts: mediaUnderstandingProviders

## Related docs

- [opencode](/providers/opencode)
---
domain: plugins
topic: "Plugin Reference Batch 2: clickclack, cloudflare-ai-gateway, codex, comfy..."
type: reference
keywords:
  - clickclack
  - cloudflare-ai-gateway
  - codex
  - comfy
  - copilot-proxy
  - deepgram
  - deepinfra
  - deepseek
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/clickclack.md
  - plugins/reference/cloudflare-ai-gateway.md
  - plugins/reference/codex.md
  - plugins/reference/comfy.md
  - plugins/reference/copilot-proxy.md
  - plugins/reference/deepgram.md
  - plugins/reference/deepinfra.md
  - plugins/reference/deepseek.md
  - plugins/reference/diagnostics-otel.md
  - plugins/reference/diagnostics-prometheus.md
  - plugins/reference/diffs.md
  - plugins/reference/discord.md
  - plugins/reference/document-extract.md
  - plugins/reference/duckduckgo.md
  - plugins/reference/elevenlabs.md
---

Plugin configuration reference entries for: clickclack, cloudflare-ai-gateway, codex, comfy, copilot-proxy, deepgram, deepinfra, deepseek.

## clickclack

# Clickclack plugin

Adds the Clickclack channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/clickclack`
- Install route: included in OpenClaw

## Surface

channels: clickclack

## Related docs

- [clickclack](/channels/clickclack)

## cloudflare-ai-gateway

# Cloudflare AI Gateway plugin

Adds Cloudflare AI Gateway model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/cloudflare-ai-gateway-provider`
- Install route: included in OpenClaw

## Surface

providers: cloudflare-ai-gateway

## Related docs

- [cloudflare-ai-gateway](/providers/cloudflare-ai-gateway)

## codex

# Codex plugin

Codex app-server harness and Codex-managed GPT model catalog.

## Distribution

- Package: `@openclaw/codex`
- Install route: npm; ClawHub

## Surface

providers: codex; contracts: mediaUnderstandingProviders, migrationProviders

## Related docs

- [codex](/plugins/codex-harness)

## comfy

# ComfyUI plugin

Adds ComfyUI model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/comfy-provider`
- Install route: included in OpenClaw

## Surface

providers: comfy; contracts: imageGenerationProviders, musicGenerationProviders, videoGenerationProviders

## Related docs

- [comfy](/providers/comfy)

## copilot-proxy

# Copilot Proxy plugin

Adds Copilot Proxy model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/copilot-proxy`
- Install route: included in OpenClaw

## Surface

providers: copilot-proxy

## deepgram

# Deepgram plugin

Adds media understanding provider support. Adds realtime transcription provider support.

## Distribution

- Package: `@openclaw/deepgram-provider`
- Install route: included in OpenClaw

## Surface

contracts: mediaUnderstandingProviders, realtimeTranscriptionProviders

## Related docs

- [deepgram](/providers/deepgram)

## deepinfra

# DeepInfra plugin

Adds DeepInfra model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/deepinfra-provider`
- Install route: included in OpenClaw

## Surface

providers: deepinfra; contracts: imageGenerationProviders, mediaUnderstandingProviders, memoryEmbeddingProviders, speechProviders, videoGenerationProviders

## Related docs

- [deepinfra](/providers/deepinfra)

## deepseek

# DeepSeek plugin

Adds DeepSeek model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/deepseek-provider`
- Install route: included in OpenClaw

## Surface

providers: deepseek

## Related docs

- [deepseek](/providers/deepseek)

## diagnostics-otel

# Diagnostics OpenTelemetry plugin

OpenClaw diagnostics OpenTelemetry exporter.

## Distribution

- Package: `@openclaw/diagnostics-otel`
- Install route: npm; ClawHub: `clawhub:@openclaw/diagnostics-otel`

## Surface

plugin

## diagnostics-prometheus

# Diagnostics Prometheus plugin

OpenClaw diagnostics Prometheus exporter.

## Distribution

- Package: `@openclaw/diagnostics-prometheus`
- Install route: npm; ClawHub: `clawhub:@openclaw/diagnostics-prometheus`

## Surface

plugin

## diffs

# Diffs plugin

Read-only diff viewer and file renderer for agents.

## Distribution

- Package: `@openclaw/diffs`
- Install route: npm; ClawHub

## Surface

contracts: tools; skills

## discord

# Discord plugin

Adds the Discord channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/discord`
- Install route: npm; ClawHub

## Surface

channels: discord

## Related docs

- [discord](/channels/discord)

## document-extract

# Document Extract plugin

Extract text and fallback page images from local document attachments.

## Distribution

- Package: `@openclaw/document-extract-plugin`
- Install route: included in OpenClaw

## Surface

contracts: documentExtractors

## Related docs

- [document-extract](/tools/pdf)

## duckduckgo

# DuckDuckGo plugin

Adds web search provider support.

## Distribution

- Package: `@openclaw/duckduckgo-plugin`
- Install route: included in OpenClaw

## Surface

contracts: webSearchProviders

## Related docs

- [duckduckgo](/tools/duckduckgo-search)

## elevenlabs

# Elevenlabs plugin

Adds media understanding provider support. Adds realtime transcription provider support. Adds text-to-speech provider support.

## Distribution

- Package: `@openclaw/elevenlabs-speech`
- Install route: included in OpenClaw

## Surface

contracts: mediaUnderstandingProviders, realtimeTranscriptionProviders, speechProviders

## Related docs

- [elevenlabs](/providers/elevenlabs)
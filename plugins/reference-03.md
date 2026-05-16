---
domain: plugins
topic: "Plugin Reference Batch 3: exa, fal, feishu, file-transfer..."
type: reference
keywords:
  - exa
  - fal
  - feishu
  - file-transfer
  - firecrawl
  - fireworks
  - github-copilot
  - google-meet
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/exa.md
  - plugins/reference/fal.md
  - plugins/reference/feishu.md
  - plugins/reference/file-transfer.md
  - plugins/reference/firecrawl.md
  - plugins/reference/fireworks.md
  - plugins/reference/github-copilot.md
  - plugins/reference/google-meet.md
  - plugins/reference/google.md
  - plugins/reference/googlechat.md
  - plugins/reference/gradium.md
  - plugins/reference/groq.md
  - plugins/reference/huggingface.md
  - plugins/reference/imessage.md
  - plugins/reference/inworld.md
---

Plugin configuration reference entries for: exa, fal, feishu, file-transfer, firecrawl, fireworks, github-copilot, google-meet.

## exa

# Exa plugin

Adds web search provider support.

## Distribution

- Package: `@openclaw/exa-plugin`
- Install route: included in OpenClaw

## Surface

contracts: webSearchProviders

## Related docs

- [exa](/tools/exa-search)

## fal

# fal plugin

Adds fal model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/fal-provider`
- Install route: included in OpenClaw

## Surface

providers: fal; contracts: imageGenerationProviders, videoGenerationProviders

## Related docs

- [fal](/providers/fal)

## feishu

# Feishu plugin

Adds the Feishu channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/feishu`
- Install route: npm; ClawHub

## Surface

channels: feishu; contracts: tools; skills

## Related docs

- [feishu](/channels/feishu)

## file-transfer

# File Transfer plugin

Fetch, list, and write files on paired nodes via dedicated node commands. Bypasses bash stdout truncation by using base64 over node.invoke for binaries up to 16 MB.

## Distribution

- Package: `@openclaw/file-transfer`
- Install route: included in OpenClaw

## Surface

contracts: tools

## firecrawl

# Firecrawl plugin

Adds agent-callable tools. Adds web fetch provider support. Adds web search provider support.

## Distribution

- Package: `@openclaw/firecrawl-plugin`
- Install route: included in OpenClaw

## Surface

contracts: tools, webFetchProviders, webSearchProviders

## Related docs

- [firecrawl](/tools/firecrawl)

## fireworks

# Fireworks plugin

Adds Fireworks model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/fireworks-provider`
- Install route: included in OpenClaw

## Surface

providers: fireworks

## Related docs

- [fireworks](/providers/fireworks)

## github-copilot

# GitHub Copilot plugin

Adds GitHub Copilot model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/github-copilot-provider`
- Install route: included in OpenClaw

## Surface

providers: github-copilot; contracts: memoryEmbeddingProviders

## Related docs

- [github-copilot](/providers/github-copilot)

## google-meet

# Google Meet plugin

Join Google Meet calls through Chrome or Twilio transports.

## Distribution

- Package: `@openclaw/google-meet`
- Install route: npm; ClawHub

## Surface

contracts: tools

## Related docs

- [google-meet](/plugins/google-meet)

## google

# Google plugin

Adds Google, Google Gemini CLI, Google Vertex model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/google-plugin`
- Install route: included in OpenClaw

## Surface

providers: google, google-gemini-cli, google-vertex; contracts: imageGenerationProviders, mediaUnderstandingProviders, memoryEmbeddingProviders, musicGenerationProviders, realtimeVoiceProviders, speechProviders, videoGenerationProviders, webSearchProviders

## Related docs

- [google](/providers/google)

## googlechat

# Google Chat plugin

Adds the Google Chat channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/googlechat`
- Install route: npm; ClawHub

## Surface

channels: googlechat

## Related docs

- [googlechat](/channels/googlechat)

## gradium

# Gradium plugin

Adds text-to-speech provider support.

## Distribution

- Package: `@openclaw/gradium-speech`
- Install route: included in OpenClaw

## Surface

contracts: speechProviders

## Related docs

- [gradium](/providers/gradium)

## groq

# Groq plugin

Adds Groq model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/groq-provider`
- Install route: included in OpenClaw

## Surface

providers: groq; contracts: mediaUnderstandingProviders

## Related docs

- [groq](/providers/groq)

## huggingface

# Hugging Face plugin

Adds Hugging Face model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/huggingface-provider`
- Install route: included in OpenClaw

## Surface

providers: huggingface

## Related docs

- [huggingface](/providers/huggingface)

## imessage

# iMessage plugin

Adds the iMessage channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/imessage`
- Install route: included in OpenClaw

## Surface

channels: imessage

## Related docs

- [imessage](/channels/imessage)

## inworld

# Inworld plugin

Inworld streaming text-to-speech (MP3, OGG_OPUS, PCM telephony).

## Distribution

- Package: `@openclaw/inworld-speech`
- Install route: included in OpenClaw

## Surface

contracts: speechProviders

## Related docs

- [inworld](/providers/inworld)
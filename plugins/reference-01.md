---
domain: plugins
topic: "Plugin Reference Batch 1: acpx, alibaba, amazon-bedrock-mantle, amazon-bedrock..."
type: reference
keywords:
  - acpx
  - alibaba
  - amazon-bedrock-mantle
  - amazon-bedrock
  - anthropic-vertex
  - anthropic
  - arcee
  - azure-speech
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/acpx.md
  - plugins/reference/alibaba.md
  - plugins/reference/amazon-bedrock-mantle.md
  - plugins/reference/amazon-bedrock.md
  - plugins/reference/anthropic-vertex.md
  - plugins/reference/anthropic.md
  - plugins/reference/arcee.md
  - plugins/reference/azure-speech.md
  - plugins/reference/bonjour.md
  - plugins/reference/brave.md
  - plugins/reference/browser.md
  - plugins/reference/byteplus.md
  - plugins/reference/canvas.md
  - plugins/reference/cerebras.md
  - plugins/reference/chutes.md
---

Plugin configuration reference entries for: acpx, alibaba, amazon-bedrock-mantle, amazon-bedrock, anthropic-vertex, anthropic, arcee, azure-speech.

## acpx

# ACPx plugin

Embedded ACP runtime backend with plugin-owned session and transport management.

## Distribution

- Package: `@openclaw/acpx`
- Install route: npm; ClawHub

## Surface

skills

## Related docs

- [acpx](/tools/acp-agents-setup)

## alibaba

# Alibaba plugin

Adds video generation provider support.

## Distribution

- Package: `@openclaw/alibaba-provider`
- Install route: included in OpenClaw

## Surface

contracts: videoGenerationProviders

## Related docs

- [alibaba](/providers/alibaba)

## amazon-bedrock-mantle

# Amazon Bedrock Mantle plugin

Adds Amazon Bedrock Mantle model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/amazon-bedrock-mantle-provider`
- Install route: npm

## Surface

providers: amazon-bedrock-mantle

## Related docs

- [amazon-bedrock-mantle](/providers/bedrock-mantle)

## amazon-bedrock

# Amazon Bedrock plugin

Adds Amazon Bedrock model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/amazon-bedrock-provider`
- Install route: npm

## Surface

providers: amazon-bedrock; contracts: memoryEmbeddingProviders

## Related docs

- [amazon-bedrock](/providers/bedrock)

## anthropic-vertex

# Anthropic Vertex plugin

Adds Anthropic Vertex model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/anthropic-vertex-provider`
- Install route: npm; ClawHub

## Surface

providers: anthropic-vertex

## anthropic

# Anthropic plugin

Adds Anthropic model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/anthropic-provider`
- Install route: included in OpenClaw

## Surface

providers: anthropic; contracts: mediaUnderstandingProviders

## Related docs

- [anthropic](/providers/anthropic)

## arcee

# Arcee plugin

Adds Arcee model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/arcee-provider`
- Install route: included in OpenClaw

## Surface

providers: arcee

## Related docs

- [arcee](/providers/arcee)

## azure-speech

# Azure Speech plugin

Azure AI Speech text-to-speech (MP3, native Ogg/Opus voice notes, PCM telephony).

## Distribution

- Package: `@openclaw/azure-speech`
- Install route: included in OpenClaw

## Surface

contracts: speechProviders

## Related docs

- [azure-speech](/providers/azure-speech)

## bonjour

# Bonjour plugin

Advertise the local OpenClaw gateway over Bonjour/mDNS.

## Distribution

- Package: `@openclaw/bonjour`
- Install route: included in OpenClaw

## Surface

plugin

## brave

# Brave plugin

Adds web search provider support.

## Distribution

- Package: `@openclaw/brave-plugin`
- Install route: npm; ClawHub

## Surface

contracts: webSearchProviders

## Related docs

- [brave](/tools/brave-search)

## browser

# Browser plugin

Adds agent-callable tools.

## Distribution

- Package: `@openclaw/browser-plugin`
- Install route: included in OpenClaw

## Surface

contracts: tools; skills

## Related docs

- [browser](/tools/browser)

## byteplus

# BytePlus plugin

Adds BytePlus, BytePlus Plan model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/byteplus-provider`
- Install route: included in OpenClaw

## Surface

providers: byteplus, byteplus-plan; contracts: videoGenerationProviders

## canvas

# Canvas plugin

Experimental Canvas control and A2UI rendering surfaces for paired nodes.

## Distribution

- Package: `@openclaw/canvas-plugin`
- Install route: included in OpenClaw

## Surface

contracts: tools

## cerebras

# Cerebras plugin

Adds Cerebras model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/cerebras-provider`
- Install route: included in OpenClaw

## Surface

providers: cerebras

## Related docs

- [cerebras](/providers/cerebras)

## chutes

# Chutes plugin

Adds Chutes model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/chutes-provider`
- Install route: included in OpenClaw

## Surface

providers: chutes

## Related docs

- [chutes](/providers/chutes)
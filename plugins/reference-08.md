---
domain: plugins
topic: "Plugin Reference Batch 8: voice-call, volcengine, voyage, vydra..."
type: reference
keywords:
  - voice-call
  - volcengine
  - voyage
  - vydra
  - web-readability
  - webhooks
  - whatsapp
  - xai
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/voice-call.md
  - plugins/reference/volcengine.md
  - plugins/reference/voyage.md
  - plugins/reference/vydra.md
  - plugins/reference/web-readability.md
  - plugins/reference/webhooks.md
  - plugins/reference/whatsapp.md
  - plugins/reference/xai.md
  - plugins/reference/xiaomi.md
  - plugins/reference/zai.md
  - plugins/reference/zalo.md
  - plugins/reference/zalouser.md
---

Plugin configuration reference entries for: voice-call, volcengine, voyage, vydra, web-readability, webhooks, whatsapp, xai.

## voice-call

# Voice Call plugin

Adds agent-callable tools.

## Distribution

- Package: `@openclaw/voice-call`
- Install route: npm; ClawHub

## Surface

contracts: tools

## Related docs

- [voice-call](/plugins/voice-call)

## volcengine

# Volcengine plugin

Adds Volcengine, Volcengine Plan model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/volcengine-provider`
- Install route: included in OpenClaw

## Surface

providers: volcengine, volcengine-plan; contracts: speechProviders

## Related docs

- [volcengine](/providers/volcengine)

## voyage

# Voyage plugin

Adds memory embedding provider support.

## Distribution

- Package: `@openclaw/voyage-provider`
- Install route: included in OpenClaw

## Surface

contracts: memoryEmbeddingProviders

## vydra

# Vydra plugin

Adds Vydra model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/vydra-provider`
- Install route: included in OpenClaw

## Surface

providers: vydra; contracts: imageGenerationProviders, speechProviders, videoGenerationProviders

## Related docs

- [vydra](/providers/vydra)

## web-readability

# Web Readability plugin

Extract readable article content from local HTML web fetch responses.

## Distribution

- Package: `@openclaw/web-readability-plugin`
- Install route: included in OpenClaw

## Surface

contracts: webContentExtractors

## webhooks

# Webhooks plugin

Authenticated inbound webhooks that bind external automation to OpenClaw TaskFlows.

## Distribution

- Package: `@openclaw/webhooks`
- Install route: included in OpenClaw

## Surface

plugin

## Related docs

- [webhooks](/plugins/webhooks)

## whatsapp

# WhatsApp plugin

Adds the WhatsApp channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/whatsapp`
- Install route: ClawHub: `clawhub:@openclaw/whatsapp`; npm

## Surface

channels: whatsapp

## Related docs

- [whatsapp](/channels/whatsapp)

## xai

# xAI plugin

Adds xAI model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/xai-plugin`
- Install route: included in OpenClaw

## Surface

providers: xai; contracts: imageGenerationProviders, mediaUnderstandingProviders, realtimeTranscriptionProviders, speechProviders, tools, videoGenerationProviders, webSearchProviders

## Related docs

- [xai](/providers/xai)

## xiaomi

# Xiaomi plugin

Adds Xiaomi model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/xiaomi-provider`
- Install route: included in OpenClaw

## Surface

providers: xiaomi; contracts: speechProviders

## Related docs

- [xiaomi](/providers/xiaomi)

## zai

# Z.AI plugin

Adds Z.AI model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/zai-provider`
- Install route: included in OpenClaw

## Surface

providers: zai; contracts: mediaUnderstandingProviders

## Related docs

- [zai](/providers/zai)

## zalo

# Zalo plugin

Adds the Zalo channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/zalo`
- Install route: npm; ClawHub

## Surface

channels: zalo

## Related docs

- [zalo](/channels/zalo)

## zalouser

# Zalo Personal plugin

Adds the Zalo Personal channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/zalouser`
- Install route: npm; ClawHub

## Surface

channels: zalouser; contracts: tools

## Related docs

- [zalouser](/channels/zalouser)
- [zalouser](/plugins/zalouser)
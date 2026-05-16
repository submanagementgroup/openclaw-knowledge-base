---
domain: plugins
topic: "Built-in Plugin Inventory"
type: reference
keywords:
  - plugin inventory
  - built-in plugins
  - plugin list
  - available plugins
  - plugin catalog
source: plugins/plugin-inventory.md
---

# Plugin inventory

This page is generated from `extensions/*/package.json`, `openclaw.plugin.json`,
and the root npm package `files` exclusions. Regenerate it with:

```bash
pnpm plugins:inventory:gen
```

## Definitions

- **Core npm package:** built into the `openclaw` npm package and available without a separate plugin install.
- **Official external package:** OpenClaw-maintained plugin omitted from the core npm package, kept in this official inventory, and installed on demand through ClawHub and/or npm.
- **Source checkout only:** repo-local plugin omitted from published npm artifacts and not advertised as an installable package.

Source checkouts are different from npm installs: after `pnpm install`, bundled
plugins load from `extensions/<id>` so local edits and package-local workspace
dependencies are available.

## Install a plugin

Use the **Distribution** column to decide whether install is needed. Plugins that
say `included in OpenClaw` are already present in the core package. Official
external packages need one install, then a Gateway restart.

For example, Discord is an official external package:

```bash
openclaw plugins install @openclaw/discord
openclaw gateway restart
openclaw plugins inspect discord --runtime --json
```

Bare package specs try ClawHub first, then npm fallback. To force a source, use
`clawhub:@openclaw/discord` or `npm:@openclaw/discord`. After install, follow
the plugin's setup doc, such as [Discord](/channels/discord), to add credentials
and channel config. See [Manage plugins](/plugins/manage-plugins) for update,
uninstall, and publishing commands.

## Core npm package

| Plugin                                                            | Description                                                                                                                                                          | Distribution                                                         | Surface                                                                                                                                                                                                                                                          |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [alibaba](/plugins/reference/alibaba)                             | Adds video generation provider support.                                                                                                                              | `@openclaw/alibaba-provider`<br />included in OpenClaw               | contracts: videoGenerationProviders                                                                                                                                                                                                                              |
| [anthropic](/plugins/reference/anthropic)                         | Adds Anthropic model provider support to OpenClaw.                                                                                                                   | `@openclaw/anthropic-provider`<br />included in OpenClaw             | providers: anthropic; contracts: mediaUnderstandingProviders                                                                                                                                                                                                     |
| [arcee](/plugins/reference/arcee)                                 | Adds Arcee model provider support to OpenClaw.                                                                                                                       | `@openclaw/arcee-provider`<br />included in OpenClaw                 | providers: arcee                                                                                                                                                                                                                                                 |
| [azure-speech](/plugins/reference/azure-speech)                   | Azure AI Speech text-to-speech (MP3, native Ogg/Opus voice notes, PCM telephony).                                                                                    | `@openclaw/azure-speech`<br />included in OpenClaw                   | contracts: speechProviders                                                                                                                                                                                                                                       |
| [bonjour](/plugins/reference/bonjour)                             | Advertise the local OpenClaw gateway over Bonjour/mDNS.                                                                                                              | `@openclaw/bonjour`<br />included in OpenClaw                        | plugin                                                                                                                                                                                                                                                           |
| [browser](/plugins/reference/browser)                             | Adds agent-callable tools.                                                                                                                                           | `@openclaw/browser-plugin`<br />included in OpenClaw                 | contracts: tools; skills                                                                                                                                                                                                                                         |
| [byteplus](/plugins/reference/byteplus)                           | Adds BytePlus, BytePlus Plan model provider support to OpenClaw.                                                                                                     | `@openclaw/byteplus-provider`<br />included in OpenClaw              | providers: byteplus, byteplus-plan; contracts: videoGenerationProviders                                                                                                                                                                                          |
| [canvas](/plugins/reference/canvas)                               | Experimental Canvas control and A2UI rendering surfaces for paired nodes.                                                                                            | `@openclaw/canvas-plugin`<br />included in OpenClaw                  | contracts: tools                                                                                                                                                                                                                                                 |
| [cerebras](/plugins/reference/cerebras)                           | Adds Cerebras model provider support to OpenClaw.                                                                                                                    | `@openclaw/cerebras-provider`<br />included in OpenClaw              | providers: cerebras                                                                                                                                                                                                                                              |
| [chutes](/plugins/reference/chutes)                               | Adds Chutes model provider support to OpenClaw.                                                                                                                      | `@openclaw/chutes-provider`<br />included in OpenClaw                | providers: chutes                                                                                                                                                                                                                                                |
| [clickclack](/plugins/reference/clickclack)                       | Adds the Clickclack channel surface for sending and receiving OpenClaw messages.                                                                                     | `@openclaw/clickclack`<br />included in OpenClaw                     | channels: clickclack                                                                                                                                                                                                                                             |
| [cloudflare-ai-gateway](/plugins/reference/cloudflare-ai-gateway) | Adds Cloudflare AI Gateway model provider support to OpenClaw.                                                                                                       | `@openclaw/cloudflare-ai-gateway-provider`<br />included in OpenClaw | providers: cloudflare-ai-gateway                                                                                                                                                                                                                                 |
| [comfy](/plugins/reference/comfy)                                 | Adds ComfyUI model provider support to OpenClaw.                                                                                                                     | `@openclaw/comfy-provider`<br />included in OpenClaw                 | providers: comfy; contracts: imageGenerationProviders, musicGenerationProviders, videoGenerationProviders                                                                                                                                                        |
| [copilot-proxy](/plugins/reference/copilot-proxy)                 | Adds Copilot Proxy model provider support to OpenClaw.                                                                                                               | `@openclaw/copilot-proxy`<br />included in OpenClaw                  | providers: copilot-proxy                                                                                                                                                                                                                                         |
| [deepgram](/plugins/reference/deepgram)                           | Adds media understanding provider support. Adds realtime transcription provider support.                                                                             | `@openclaw/deepgram-provider`<br />included in OpenClaw              | contracts: mediaUnderstandingProviders, realtimeTranscriptionProviders                                                                                                                                                                                           |
| [deepinfra](/plugins/reference/deepinfra)                         | Adds DeepInfra model provider support to OpenClaw.                                                                                                                   | `@openclaw/deepinfra-provider`<br />included in OpenClaw             | providers: deepinfra; contracts: imageGenerationProviders, mediaUnderstandingProviders, memoryEmbeddingProviders, speechProviders, videoGenerationProviders                                                                                                      |
| [deepseek](/plugins/reference/deepseek)                           | Adds DeepSeek model provider support to OpenClaw.                                                                                                                    | `@openclaw/deepseek-provider`<br />included in OpenClaw              | providers: deepseek                                                                                                                                                                                                                                              |
| [document-extract](/plugins/reference/document-extract)           | Extract text and fallback page images from local document attachments.                                                                                               | `@openclaw/document-extract-plugin`<br />included in OpenClaw        | contracts: documentExtractors                                                                                                                                                                                                                                    |
| [duckduckgo](/plugins/reference/duckduckgo)                       | Adds web search provider support.                                                                                                                                    | `@openclaw/duckduckgo-plugin`<br />included in OpenClaw              | contracts: webSearchProviders                                                                                                                                                                                                                                    |
| [elevenlabs](/plugins/reference/elevenlabs)                       | Adds media understanding provider support. Adds realtime transcription provider support. Adds text-to-speech provider support.                                       | `@openclaw/elevenlabs-speech`<br />included in OpenClaw              | contracts: mediaUnderstandingProviders, realtimeTranscriptionProviders, speechProviders                                                                
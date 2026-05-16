---
domain: plugins
topic: "Plugin Reference Batch 7: slack, stepfun, synology-chat, synthetic..."
type: reference
keywords:
  - slack
  - stepfun
  - synology-chat
  - synthetic
  - tavily
  - telegram
  - tencent
  - tlon
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/slack.md
  - plugins/reference/stepfun.md
  - plugins/reference/synology-chat.md
  - plugins/reference/synthetic.md
  - plugins/reference/tavily.md
  - plugins/reference/telegram.md
  - plugins/reference/tencent.md
  - plugins/reference/tlon.md
  - plugins/reference/together.md
  - plugins/reference/tokenjuice.md
  - plugins/reference/tts-local-cli.md
  - plugins/reference/twitch.md
  - plugins/reference/venice.md
  - plugins/reference/vercel-ai-gateway.md
  - plugins/reference/vllm.md
---

Plugin configuration reference entries for: slack, stepfun, synology-chat, synthetic, tavily, telegram, tencent, tlon.

## slack

# Slack plugin

Adds the Slack channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/slack`
- Install route: npm; ClawHub

## Surface

channels: slack

## Related docs

- [slack](/channels/slack)

## stepfun

# StepFun plugin

Adds StepFun, StepFun Plan model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/stepfun-provider`
- Install route: included in OpenClaw

## Surface

providers: stepfun, stepfun-plan

## Related docs

- [stepfun](/providers/stepfun)

## synology-chat

# Synology Chat plugin

Adds the Synology Chat channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/synology-chat`
- Install route: npm; ClawHub

## Surface

channels: synology-chat

## Related docs

- [synology-chat](/channels/synology-chat)

## synthetic

# Synthetic plugin

Adds Synthetic model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/synthetic-provider`
- Install route: included in OpenClaw

## Surface

providers: synthetic

## Related docs

- [synthetic](/providers/synthetic)

## tavily

# Tavily plugin

Adds agent-callable tools. Adds web search provider support.

## Distribution

- Package: `@openclaw/tavily-plugin`
- Install route: included in OpenClaw

## Surface

contracts: tools, webSearchProviders; skills

## Related docs

- [tavily](/tools/tavily)

## telegram

# Telegram plugin

Adds the Telegram channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/telegram`
- Install route: included in OpenClaw

## Surface

channels: telegram

## Related docs

- [telegram](/channels/telegram)

## tencent

# Tencent plugin

Adds Tencent TokenHub model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/tencent-provider`
- Install route: included in OpenClaw

## Surface

providers: tencent-tokenhub

## Related docs

- [tencent](/providers/tencent)

## tlon

# Tlon plugin

Adds the Tlon channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/tlon`
- Install route: npm; ClawHub

## Surface

channels: tlon; contracts: tools; skills

## Related docs

- [tlon](/channels/tlon)

## together

# Together plugin

Adds Together model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/together-provider`
- Install route: included in OpenClaw

## Surface

providers: together; contracts: videoGenerationProviders

## Related docs

- [together](/providers/together)

## tokenjuice

# Tokenjuice plugin

Compacts exec and bash tool results with tokenjuice reducers.

## Distribution

- Package: `@openclaw/tokenjuice`
- Install route: included in OpenClaw

## Surface

contracts: agentToolResultMiddleware

## Related docs

- [tokenjuice](/tools/tokenjuice)

## tts-local-cli

# TTS Local CLI plugin

Adds text-to-speech provider support.

## Distribution

- Package: `@openclaw/tts-local-cli`
- Install route: included in OpenClaw

## Surface

contracts: speechProviders

## twitch

# Twitch plugin

Adds the Twitch channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/twitch`
- Install route: npm; ClawHub

## Surface

channels: twitch

## Related docs

- [twitch](/channels/twitch)

## venice

# Venice plugin

Adds Venice model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/venice-provider`
- Install route: included in OpenClaw

## Surface

providers: venice

## Related docs

- [venice](/providers/venice)

## vercel-ai-gateway

# Vercel AI Gateway plugin

Adds Vercel AI Gateway model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/vercel-ai-gateway-provider`
- Install route: included in OpenClaw

## Surface

providers: vercel-ai-gateway

## Related docs

- [vercel-ai-gateway](/providers/vercel-ai-gateway)

## vllm

# vLLM plugin

Adds vLLM model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/vllm-provider`
- Install route: included in OpenClaw

## Surface

providers: vllm

## Related docs

- [vllm](/providers/vllm)
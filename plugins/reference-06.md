---
domain: plugins
topic: "Plugin Reference Batch 6: openrouter, openshell, perplexity, qa-channel..."
type: reference
keywords:
  - openrouter
  - openshell
  - perplexity
  - qa-channel
  - qa-lab
  - qa-matrix
  - qianfan
  - qqbot
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/openrouter.md
  - plugins/reference/openshell.md
  - plugins/reference/perplexity.md
  - plugins/reference/qa-channel.md
  - plugins/reference/qa-lab.md
  - plugins/reference/qa-matrix.md
  - plugins/reference/qianfan.md
  - plugins/reference/qqbot.md
  - plugins/reference/qwen.md
  - plugins/reference/runway.md
  - plugins/reference/searxng.md
  - plugins/reference/senseaudio.md
  - plugins/reference/sglang.md
  - plugins/reference/signal.md
  - plugins/reference/skill-workshop.md
---

Plugin configuration reference entries for: openrouter, openshell, perplexity, qa-channel, qa-lab, qa-matrix, qianfan, qqbot.

## openrouter

# OpenRouter plugin

Adds OpenRouter model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/openrouter-provider`
- Install route: included in OpenClaw

## Surface

providers: openrouter; contracts: imageGenerationProviders, mediaUnderstandingProviders, speechProviders, videoGenerationProviders

## Related docs

- [openrouter](/providers/openrouter)

## openshell

# Openshell plugin

Sandbox backend powered by OpenShell with mirrored local workspaces and SSH-based command execution.

## Distribution

- Package: `@openclaw/openshell-sandbox`
- Install route: npm; ClawHub

## Surface

plugin

## perplexity

# Perplexity plugin

Adds web search provider support.

## Distribution

- Package: `@openclaw/perplexity-plugin`
- Install route: included in OpenClaw

## Surface

contracts: webSearchProviders

## Related docs

- [perplexity](/tools/perplexity-search)

## qa-channel

# QA Channel plugin

Adds the QA Channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/qa-channel`
- Install route: source checkout only

## Surface

channels: qa-channel

## Related docs

- [qa-channel](/channels/qa-channel)

## qa-lab

# QA Lab plugin

OpenClaw QA lab plugin with private debugger UI and scenario runner.

## Distribution

- Package: `@openclaw/qa-lab`
- Install route: source checkout only

## Surface

plugin

## qa-matrix

# QA Matrix plugin

Matrix QA transport runner and substrate.

## Distribution

- Package: `@openclaw/qa-matrix`
- Install route: source checkout only

## Surface

plugin

## qianfan

# Qianfan plugin

Adds Qianfan model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/qianfan-provider`
- Install route: included in OpenClaw

## Surface

providers: qianfan

## Related docs

- [qianfan](/providers/qianfan)

## qqbot

# QQ Bot plugin

Adds the QQ Bot channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/qqbot`
- Install route: npm; ClawHub

## Surface

channels: qqbot; contracts: tools; skills

## Related docs

- [qqbot](/channels/qqbot)

## qwen

# Qwen plugin

Adds Qwen, Qwen Cloud, Model Studio, DashScope model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/qwen-provider`
- Install route: included in OpenClaw

## Surface

providers: qwen, qwencloud, modelstudio, dashscope; contracts: mediaUnderstandingProviders, videoGenerationProviders

## Related docs

- [qwen](/providers/qwen)

## runway

# Runway plugin

Adds video generation provider support.

## Distribution

- Package: `@openclaw/runway-provider`
- Install route: included in OpenClaw

## Surface

contracts: videoGenerationProviders

## Related docs

- [runway](/providers/runway)

## searxng

# SearXNG plugin

Adds web search provider support.

## Distribution

- Package: `@openclaw/searxng-plugin`
- Install route: included in OpenClaw

## Surface

contracts: webSearchProviders

## senseaudio

# Senseaudio plugin

Adds media understanding provider support.

## Distribution

- Package: `@openclaw/senseaudio-provider`
- Install route: included in OpenClaw

## Surface

contracts: mediaUnderstandingProviders

## Related docs

- [senseaudio](/providers/senseaudio)

## sglang

# SGLang plugin

Adds SGLang model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/sglang-provider`
- Install route: included in OpenClaw

## Surface

providers: sglang

## Related docs

- [sglang](/providers/sglang)

## signal

# Signal plugin

Adds the Signal channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/signal`
- Install route: included in OpenClaw

## Surface

channels: signal

## Related docs

- [signal](/channels/signal)

## skill-workshop

# Skill Workshop plugin

Captures repeatable workflows as workspace skills, with pending review, safe writes, and skill prompt refresh.

## Distribution

- Package: `@openclaw/skill-workshop`
- Install route: included in OpenClaw

## Surface

contracts: tools

## Related docs

- [skill-workshop](/plugins/skill-workshop)
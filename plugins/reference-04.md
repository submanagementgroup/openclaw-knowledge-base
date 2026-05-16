---
domain: plugins
topic: "Plugin Reference Batch 4: irc, kilocode, kimi, line..."
type: reference
keywords:
  - irc
  - kilocode
  - kimi
  - line
  - litellm
  - llm-task
  - lmstudio
  - lobster
  - plugin reference
  - plugin config
  - plugin packages
source: 
  - plugins/reference/irc.md
  - plugins/reference/kilocode.md
  - plugins/reference/kimi.md
  - plugins/reference/line.md
  - plugins/reference/litellm.md
  - plugins/reference/llm-task.md
  - plugins/reference/lmstudio.md
  - plugins/reference/lobster.md
  - plugins/reference/matrix.md
  - plugins/reference/mattermost.md
  - plugins/reference/memory-core.md
  - plugins/reference/memory-lancedb.md
  - plugins/reference/memory-wiki.md
  - plugins/reference/microsoft-foundry.md
  - plugins/reference/microsoft.md
---

Plugin configuration reference entries for: irc, kilocode, kimi, line, litellm, llm-task, lmstudio, lobster.

## irc

# IRC plugin

Adds the IRC channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/irc`
- Install route: included in OpenClaw

## Surface

channels: irc

## Related docs

- [irc](/channels/irc)

## kilocode

# Kilocode plugin

Adds Kilocode model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/kilocode-provider`
- Install route: included in OpenClaw

## Surface

providers: kilocode

## Related docs

- [kilocode](/providers/kilocode)

## kimi

# Kimi plugin

Adds Kimi, Kimi Coding model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/kimi-provider`
- Install route: included in OpenClaw

## Surface

providers: kimi, kimi-coding

## Related docs

- [kimi](/providers/moonshot)

## line

# LINE plugin

Adds the LINE channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/line`
- Install route: npm; ClawHub

## Surface

channels: line

## Related docs

- [line](/channels/line)

## litellm

# LiteLLM plugin

Adds LiteLLM model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/litellm-provider`
- Install route: included in OpenClaw

## Surface

providers: litellm; contracts: imageGenerationProviders

## Related docs

- [litellm](/providers/litellm)

## llm-task

# LLM Task plugin

Generic JSON-only LLM tool for structured tasks callable from workflows.

## Distribution

- Package: `@openclaw/llm-task`
- Install route: included in OpenClaw

## Surface

contracts: tools

## lmstudio

# LM Studio plugin

Adds LM Studio model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/lmstudio-provider`
- Install route: included in OpenClaw

## Surface

providers: lmstudio; contracts: memoryEmbeddingProviders

## Related docs

- [lmstudio](/providers/lmstudio)

## lobster

# Lobster plugin

Typed workflow tool with resumable approvals.

## Distribution

- Package: `@openclaw/lobster`
- Install route: npm; ClawHub

## Surface

contracts: tools

## matrix

# Matrix plugin

Adds the Matrix channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/matrix`
- Install route: ClawHub: `clawhub:@openclaw/matrix`; npm

## Surface

channels: matrix

## Related docs

- [matrix](/channels/matrix)

## mattermost

# Mattermost plugin

Adds the Mattermost channel surface for sending and receiving OpenClaw messages.

## Distribution

- Package: `@openclaw/mattermost`
- Install route: included in OpenClaw

## Surface

channels: mattermost

## Related docs

- [mattermost](/channels/mattermost)

## memory-core

# Memory Core plugin

Adds memory embedding provider support. Adds agent-callable tools.

## Distribution

- Package: `@openclaw/memory-core`
- Install route: included in OpenClaw

## Surface

contracts: memoryEmbeddingProviders, tools

## memory-lancedb

# Memory Lancedb plugin

Adds agent-callable tools.

## Distribution

- Package: `@openclaw/memory-lancedb`
- Install route: npm; ClawHub

## Surface

contracts: tools

## Related docs

- [memory-lancedb](/plugins/memory-lancedb)

## memory-wiki

# Memory Wiki plugin

Persistent wiki compiler and Obsidian-friendly knowledge vault for OpenClaw.

## Distribution

- Package: `@openclaw/memory-wiki`
- Install route: included in OpenClaw

## Surface

contracts: tools; skills

## Related docs

- [memory-wiki](/plugins/memory-wiki)

## microsoft-foundry

# Microsoft Foundry plugin

Adds Microsoft Foundry model provider support to OpenClaw.

## Distribution

- Package: `@openclaw/microsoft-foundry`
- Install route: included in OpenClaw

## Surface

providers: microsoft-foundry

## microsoft

# Microsoft plugin

Adds text-to-speech provider support.

## Distribution

- Package: `@openclaw/microsoft-speech`
- Install route: included in OpenClaw

## Surface

contracts: speechProviders
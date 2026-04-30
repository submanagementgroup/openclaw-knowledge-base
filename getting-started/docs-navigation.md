---
domain: getting-started
topic: "OpenClaw Documentation Map and Use Case Navigation"
type: reference
keywords:
  - docs navigation
  - documentation map
  - hubs
  - showcase
  - use cases
  - getting started guide
related:
  - getting-started/quickstart
  - getting-started/what-is-openclaw
source:
  - start/hubs.md
  - start/showcase.md
---

OpenClaw documentation is organized into several major areas. This file maps the major use cases to the relevant documentation sections.

## Documentation Areas

| Area | Description |
|------|-------------|
| `getting-started/` | Installation, quickstart, wizard, onboarding |
| `gateway/` | Gateway configuration, operation, secrets, sandboxing |
| `channels/` | Per-channel setup: Telegram, WhatsApp, Discord, Slack, etc. |
| `providers/` | AI provider setup: OpenAI, Anthropic, Ollama, Google, etc. |
| `tools/` | Agent tools: browser, exec, search, skills, media |
| `concepts/` | Core concepts: memory, sessions, agent loop, multi-agent |
| `automation/` | Cron, heartbeat, hooks, tasks, standing orders |
| `plugins/` | Plugin development: SDK, manifest, architecture |
| `install/` | Platform-specific install: Docker, Kubernetes, VPS, cloud |
| `platforms/` | OS-specific: macOS app, iOS, Android, Raspberry Pi |
| `security/` | Threat model, audit, network proxy |
| `memory/` | Memory search, QMD, builtin engine, active memory |
| `cli/` | CLI command reference |
| `reference/` | API, templates, config reference |

## Common Starting Points

**First time here:** See `getting-started/quickstart.md`

**Connecting a channel:** See `channels/<channel-name>.md`

**Choosing a model provider:** See `providers/<provider-name>.md`

**Scheduling tasks:** See `automation/cron-jobs.md`

**Plugin development:** See `plugins/sdk-overview.md`

If you are new to OpenClaw, start with [Getting Started](/start/getting-started).

Use these hubs to discover every page, including deep dives and reference docs that don’t appear in the left nav.

## Start here

- [Index](/)
- [Getting Started](/start/getting-started)
- [Onboarding](/start/onboarding)
- [Onboarding (CLI)](/start/wizard)
- [Setup](/start/setup)
- [Dashboard (local Gateway)](http://127.0.0.1:18789/)
- [Help](/help)
- [Docs directory](/start/docs-directory)
- [Configuration](/gateway/configuration)
- [Configuration examples](/gateway/configuration-examples)
- [OpenClaw assistant](/start/openclaw)
- [Showcase](/start/showcase)
- [Lore](/start/lore)

## Installation + updates

- [Docker](/install/docker)
- [Nix](/install/nix)
- [Updating / rollback](/install/updating)
- [Bun workflow (experimental)](/install/bun)

## Core concepts

- [Architecture](/concepts/architecture)
- [Features](/concepts/features)
- [Network hub](/network)
- [Agent runtime](/concepts/agent)
- [Agent workspace](/concepts/agent-workspace)
- [Memory](/concepts/memory)
- [Agent loop](/concepts/agent-loop)
- [Streaming + chunking](/concepts/streaming)
- [Multi-agent routing](/concepts/multi-agent)
- [Compaction](/concepts/compaction)
- [Sessions](/concepts/session)
- [Session pruning](/concepts/session-pruning)
- [Session tools](/concepts/session-tool)
- [Queue](/concepts/queue)
- [Slash commands](/tools/slash-commands)
- [RPC adapters](/reference/rpc)
- [TypeBox schemas](/concepts/typebox)
- [Timezone handling](/concepts/timezone)
- [Presence](/concepts/presence)
- [Discovery + transports](/gateway/discovery)
- [Bonjour](/gateway/bonjour)
- [Channel routing](/channels/channel-routing)
- [Groups](/channels/groups)
- [Group messages](/channels/group-messages)
- [Model failover](/concepts/model-failover)
- [OAuth](/concepts/oauth)

## Providers + ingress

- [Chat channels hub](/channels)
- [Model providers hub](/providers/models)
- [WhatsApp](/channels/whatsapp)
- [Telegram](/channels/telegram)
- [Slack](/channels/slack)
- [Discord](/channels/discord)
- [Mattermost](/channels/mattermost)
- [Signal](/channels/signal)
- [BlueBubbles (iMessage)](/channels/bluebubbles)
- [QQ Bot](/channels/qqbot)
- [iMessage (legacy)](/channels/imessage)
- [Location parsing](/channels/location)
- [WebChat](/web/webchat)
- [Webhooks](/automation/cron-jobs#webhooks)
- [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration)

## Gateway + operations

- [Gateway runbook](/gateway)
- [Network model](/gateway/network-model)
- [Gateway pairing](/gateway/pairing)
- [Gateway lock](/gateway/gateway-lock)
- [Background process](/gateway/background-process)
- [Health](/gateway/health)
- [Heartbeat](/gateway/heartbeat)
- [Doctor](/gateway/doctor)
- [Logging](/gateway/logging)
- [Sandboxing](/gateway/sandboxing)
- [Dashboard](/web/dashboard)
- [Control UI](/web/control-ui)
- [Remote access](/gateway/remote)
- [Remote gateway README](/gateway/remote-gateway-readme)
- [Tailscale](/gateway/tailsca

## Showcase: What People Build with OpenClaw

OpenClaw projects are not toy demos. People are shipping PR review loops, mobile apps, home automation, voice systems, devtools, and memory-heavy workflows from the channels they already use — chat-native builds on Telegram, WhatsApp, Discord, and terminals; real automation for booking, shopping, and support without waiting for an API; and physical-world integrations with printers, vacuums, cameras, and home systems.

**Want to be featured?** Share your project in [#self-promotion on Discord](https://discord.gg/clawd) or [tag @openclaw on X](https://x.com/openclaw).

## Videos

Start here if you want the shortest path from "what is this?" to "okay, I get it."

  VelvetShark, 28 minutes. Install, onboard, and get to a first working assistant end to end.

  A faster pass across real projects, surfaces, and workflows built around OpenClaw.

  Examples from the community, from chat-native coding loops to hardware and personal automation.

## Fresh from Discord

Recent standouts across coding, devtools, mobile, and chat-native product building.

  **@bangnokia** • `review` `github` `telegram`

OpenCode finishes the change, opens a PR, OpenClaw reviews the diff and replies in Telegram with suggestions plus a clear merge verdict.

  **@prades_maxime** • `skills` `local` `csv`

Asked "Robby" (@openclaw) for a local wine cellar skill. It requests a sample CSV export and a store path, then builds and tests the skill (962 bottles in the example).

  **@marchattonhere** • `automation` `browser` `shopping`

Weekly meal plan, regulars, book delivery slot, confirm order. No APIs, just browser control.

  **@am-will** • `devtools` `screenshots` `markdown`

Hotkey a screen region, Gemini vision, instant Markdown in your clipboard.

  **@kitze** • `ui` `skills` `sync`

Desktop app to manage skills and commands across Agents, Claude, Codex, and OpenClaw.

  **Community** • `voice` `tts` `telegram`

Wraps papla.media TTS and sends results as Telegram voice notes (no annoying autoplay).

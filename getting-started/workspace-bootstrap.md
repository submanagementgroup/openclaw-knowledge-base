---
domain: getting-started
topic: "Agent Workspace Bootstrap Files: AGENTS.md, SOUL.md, MEMORY.md Auto-Injection"
type: concept
keywords:
  - workspace bootstrap
  - AGENTS.md
  - SOUL.md
  - MEMORY.md
  - HEARTBEAT.md
  - context injection
  - bootstrap files
related:
  - concepts/agent-workspace
  - concepts/system-prompt
  - concepts/memory
source:
  - start/bootstrapping.md
  - start/docs-directory.md
---

OpenClaw auto-injects a set of workspace files into every agent session as bootstrap context. Understanding these files is essential for configuring what the agent knows at startup.

## Auto-Injected Workspace Files

Every session loads these files from `~/.openclaw/workspace/` (or `agents.defaults.workspace`):

| File | Purpose |
|------|---------|
| `AGENTS.md` | Primary agent instructions, standing orders, capabilities |
| `SOUL.md` | Agent personality, tone, and style |
| `TOOLS.md` | Tool usage guidance and policies |
| `IDENTITY.md` | Agent identity and role definition |
| `USER.md` | User-specific preferences and context |
| `HEARTBEAT.md` | Heartbeat task list and periodic goals |
| `BOOTSTRAP.md` | Supplemental per-session context |
| `MEMORY.md` | Long-term memory (durable facts and preferences) |

Files in subdirectories are **not** auto-injected unless explicitly listed in `agents.defaults.contextInjection`.

## Docs Directory and Agent Instruction Files

The docs directory (`~/.openclaw/docs/` or the workspace `docs/` subdirectory) holds agent instruction files (AGENTS.md, CLAUDE.md) that tell coding agents about the project. These are separate from gateway bootstrap files.

Bootstrapping is the **first‑run** ritual that prepares an agent workspace and
collects identity details. It happens after onboarding, when the agent starts
for the first time.

## What bootstrapping does

On the first agent run, OpenClaw bootstraps the workspace (default
`~/.openclaw/workspace`):

- Seeds `AGENTS.md`, `BOOTSTRAP.md`, `IDENTITY.md`, `USER.md`.
- Runs a short Q&A ritual (one question at a time).
- Writes identity + preferences to `IDENTITY.md`, `USER.md`, `SOUL.md`.
- Removes `BOOTSTRAP.md` when finished so it only runs once.

For embedded/local model runs, OpenClaw keeps `BOOTSTRAP.md` out of the
privileged system context. On the primary interactive first run, it still passes
the file contents in the user prompt so models that do not reliably call the
`read` tool can complete the ritual. If the current run cannot safely access the
workspace, the agent gets a limited bootstrap note instead of a generic greeting.

## Skipping bootstrapping

To skip this for a pre-seeded workspace, run `openclaw onboard --skip-bootstrap`.

## Where it runs

Bootstrapping always runs on the **gateway host**. If the macOS app connects to
a remote Gateway, the workspace and bootstrapping files live on that remote
machine.

When the Gateway runs on another machine, edit workspace files on the gateway
host (for example, `user@gateway-host:~/.openclaw/workspace`).

## Related docs

- macOS app onboarding: [Onboarding](/start/onboarding)
- Workspace layout: [Agent workspace](/concepts/agent-workspace)

This page is a curated index. If you are new, start with [Getting Started](/start/getting-started).
For a complete map of the docs, see [Docs hubs](/start/hubs).

## Start here

- [Docs hubs (all pages linked)](/start/hubs)
- [Help](/help)
- [Configuration](/gateway/configuration)
- [Configuration examples](/gateway/configuration-examples)
- [Slash commands](/tools/slash-commands)
- [Multi-agent routing](/concepts/multi-agent)
- [Updating and rollback](/install/updating)
- [Pairing (DM and nodes)](/channels/pairing)
- [Nix mode](/install/nix)
- [OpenClaw assistant setup](/start/openclaw)
- [Skills](/tools/skills)
- [Skills config](/tools/skills-config)
- [Workspace templates](/reference/templates/AGENTS)
- [RPC adapters](/reference/rpc)
- [Gateway runbook](/gateway)
- [Nodes (iOS and Android)](/nodes)
- [Web surfaces (Control UI)](/web)
- [Discovery and transports](/gateway/discovery)
- [Remote access](/gateway/remote)

## Providers and UX

- [WebChat](/web/webchat)
- [Control UI (browser)](/web/control-ui)
- [Telegram](/channels/telegram)
- [Discord](/channels/discord)
- [Mattermost](/channels/mattermost)
- [BlueBubbles (iMessage)](/channels/bluebubbles)
- [QQ Bot](/channels/qqbot)
- [iMessage (legacy)](/channels/imessage)
- [Groups](/channels/groups)
- [WhatsApp group messages](/channels/group-messages)
- [Media images](/nodes/images)
- [Media audio](/nodes/audio)

## Companion apps

- [macOS app](/platforms/macos)
- [iOS app](/platforms/ios)
- [Android app](/platforms/android)
- [Windows (WSL2)](/platforms/windows)
- [Linux app](/platforms/linux)

## Operations and safety

- [Sessions](/concepts/session)
- [Cron jobs](/automation/cron-jobs)
- [Webhooks](/automation/cron-jobs#webhooks)
- [Gmail hooks (Pub/Sub)](/automation/cron-jobs#gmail-pubsub-integration)
- [Security](/gateway/security)
- [Troubleshooting](/gateway/troubleshooting)

## Related

- [Getting started](/start/getting-started)
- [Docs hubs](/start/hubs)

## Controlling Bootstrap Content

Configure which files are injected via `agents.defaults.contextInjection` in `openclaw.json`:

```json5
{
  agents: {
    defaults: {
      contextInjection: {
        workspace: true,           // inject workspace files
        memory: true,              // inject MEMORY.md
        dailyNotes: true,          // inject today's/yesterday's notes
      }
    }
  }
}
```

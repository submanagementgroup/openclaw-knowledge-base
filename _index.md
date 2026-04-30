---
domain: _meta
topic: "Knowledge Base Index"
type: reference
keywords:
  - index
  - manifest
  - table of contents
  - file list
source: auto-generated
---

# OpenClaw Knowledge Base Index

Built from 533 source documentation files. Covers all OpenClaw domains.
Rebuild date: April 30, 2026.

## Domains

| Domain | Files | Description |
|--------|-------|-------------|
| `automation/` | 6 | Cron jobs, heartbeat, hooks, tasks, webhooks, and standing orders |
| `channels/` | 32 | Messaging channel setup: Telegram, WhatsApp, Discord, Slack, Matrix, Teams, and 20+ more |
| `cli/` | 30 | CLI command reference for all openclaw subcommands |
| `concepts/` | 29 | Core concepts: agent loop, memory, sessions, models, context, multi-agent routing |
| `gateway/` | 23 | Gateway service: configuration, security, secrets, sandboxing, protocols, APIs, and CLI backends |
| `getting-started/` | 7 | Quickstart, installation, onboarding wizard, workspace bootstrap, and CLI automation |
| `install/` | 14 | Platform-specific installation: Docker, Kubernetes, cloud VPS, Nix, and package managers |
| `internals/` | 6 | Internals: Pi integration, CI pipeline, QA automation, design plans, and release process |
| `memory/` | 7 | Memory system: search, QMD engine, active memory, LanceDB, and memory wiki |
| `nodes/` | 6 | iOS, Android, and macOS nodes: camera, audio, voice wake, media, location, troubleshooting |
| `platforms/` | 10 | Platform-specific guides: macOS app, iOS, Android, Linux, Windows, VPS, DigitalOcean, Oracle |
| `plugins/` | 32 | Plugin SDK, architecture, building channels/providers/tools, and bundled plugins |
| `providers/` | 58 | AI provider setup: OpenAI, Anthropic, Google, Ollama, Z.AI, and 50+ more |
| `reference/` | 19 | API reference, templates, token usage, prompt caching, and SDK design |
| `security/` | 5 | Threat model, security audit, network proxy, and formal verification |
| `tools/` | 36 | Agent tools: exec, browser, search, TTS, image/video generation, skills, search providers |
| `troubleshooting/` | 15 | Troubleshooting: FAQ, WSL2 browser, first-run issues, model errors, debugging, and testing |
| `web-ui/` | 2 | Web interfaces: Control UI dashboard, WebChat, and terminal TUI |

**Total: 347 KB files across 18 domains**

## New Files Added (This Rebuild)

### cli/ — 14 new files

| File | Topic |
|------|-------|
| `cli/clawbot-cmd.md` | openclaw clawbot: Legacy Alias Namespace |
| `cli/commitments-cmd.md` | openclaw commitments: Inferred Follow-Up Commitments |
| `cli/completion-cmd.md` | openclaw completion: Shell Completion Scripts |
| `cli/configure-cmd.md` | openclaw configure: Interactive Configuration Wizard |
| `cli/daemon-cmd.md` | openclaw daemon: Legacy Gateway Service Alias |
| `cli/directory-cmd.md` | openclaw directory: Channel Directory Lookups |
| `cli/docs-cmd.md` | openclaw docs: Search the Live Docs Index |
| `cli/migrate-cmd.md` | openclaw migrate: Import State from Another Agent System |
| `cli/nodes-cmd.md` | openclaw nodes: Manage Paired Nodes and Capabilities |
| `cli/proxy-cmd.md` | openclaw proxy: Local Debug Proxy and Traffic Capture |
| `cli/setup-cmd.md` | openclaw setup: Initialize Config and Workspace |
| `cli/system-cmd.md` | openclaw system: System Events, Heartbeat, and Presence |
| `cli/uninstall-cmd.md` | openclaw uninstall: Remove Gateway Service and Local Data |
| `cli/voicecall-cmd.md` | openclaw voicecall: Voice-Call Plugin CLI Commands |

### tools/ — 7 new files

| File | Topic |
|------|-------|
| `tools/duckduckgo-search.md` | DuckDuckGo Web Search: Key-Free Fallback Provider |
| `tools/firecrawl.md` | Firecrawl: Search, Scrape, and web_fetch Fallback |
| `tools/kimi-search.md` | Kimi Web Search: AI-Synthesized Answers via Moonshot |
| `tools/minimax-search.md` | MiniMax Web Search: Coding Plan Search API |
| `tools/ollama-search.md` | Ollama Web Search: Local or Hosted Search |
| `tools/tavily.md` | Tavily: Search and Extract Tools |
| `tools/tokenjuice.md` | Tokenjuice: Compact Noisy Exec and Bash Tool Results |

### platforms/ — 5 new files

| File | Topic |
|------|-------|
| `platforms/digitalocean.md` | DigitalOcean: Running OpenClaw on a Droplet |
| `platforms/macos-app-features.md` | macOS App Features: Icon, Logging, Remote, Skills, Voice Wake |
| `platforms/macos-gateway-lifecycle.md` | macOS Gateway Lifecycle: launchd, Health, and WebChat |
| `platforms/oracle.md` | Oracle Cloud: Always Free ARM ($0/month) |
| `platforms/platforms-overview.md` | Platforms Overview: OS Support and Gateway Service Install |

### install/ — 4 new files

| File | Topic |
|------|-------|
| `install/docker-vm-runtime.md` | Docker VM Runtime: Shared Steps for Cloud VM Deployments |
| `install/hetzner.md` | Hetzner: Production VPS Deployment with Docker |
| `install/hostinger.md` | Hostinger: 1-Click Managed or VPS Docker Install |
| `install/nix.md` | Nix: Declarative Install with Home Manager |

### nodes/ — 4 new files

| File | Topic |
|------|-------|
| `nodes/images.md` | Image and Media Support: Send and Reply Pipeline |
| `nodes/location-command.md` | Location Command: node.invoke location.get |
| `nodes/troubleshooting.md` | Node Troubleshooting: Pairing, Permissions, Error Codes |
| `nodes/voicewake.md` | Voice Wake: Global Wake Words and Routing |

### gateway/ — 2 new files

| File | Topic |
|------|-------|
| `gateway/bridge-protocol.md` | Bridge Protocol: Historical TCP Bridge (Removed) |
| `gateway/cli-backends.md` | CLI Backends: Local AI CLI Fallback with MCP Bridge |

### getting-started/ — 2 new files

| File | Topic |
|------|-------|
| `getting-started/cli-automation.md` | CLI Automation: Scripted Non-Interactive Onboarding |
| `getting-started/onboarding-overview.md` | Onboarding Overview: CLI Wizard vs macOS App |

### providers/ — 1 new file

| File | Topic |
|------|-------|
| `providers/zai.md` | Z.AI: GLM Model Provider Setup and Configuration |

### troubleshooting/ — 1 new file

| File | Topic |
|------|-------|
| `troubleshooting/browser-wsl2-cdp.md` | WSL2 + Windows + Remote Chrome CDP Troubleshooting |

## Skipped / Redirect-Only Sources

| Source | Reason |
|--------|--------|
| `tts.md` | Pure redirect to `/tools/tts` — no content |
| `automation/cron-vs-heartbeat.md` | Pure redirect to `/automation` — no content |
| `automation/poll.md` | Pure redirect to `/cli/message` — no content |
| `automation/troubleshooting.md` | Pure redirect to `/automation/cron-jobs#troubleshooting` — no content |

## Already Merged Sources (Dev Templates)

| Source | Merged Into |
|--------|------------|
| `reference/templates/AGENTS.dev.md` | `reference/workspace-templates.md` (already in source list) |
| `reference/templates/SOUL.dev.md` | `reference/workspace-templates.md` (already in source list) |
| `reference/templates/TOOLS.dev.md` | `reference/workspace-templates-dev.md` (already in source list) |
| `reference/templates/IDENTITY.dev.md` | `reference/workspace-templates-dev.md` (already in source list) |
| `reference/templates/USER.dev.md` | `reference/workspace-templates-dev.md` (already in source list) |

## All Files

| File | Type | Topic |
|------|------|-------|
| `automation/automation-misc.md` | reference | Automation: Auth Monitoring, ClawFlow, Polling, Webhooks, Gmail PubSub, and Troubleshooting |
| `automation/automation-overview.md` | concept | Automation Overview: Cron, Heartbeat, Hooks, Webhooks, Tasks, and Standing Orders |
| `automation/cron-jobs.md` | procedure | Cron Jobs: Scheduling Periodic Agent Runs, One-Shot Jobs, and Webhook Triggers |
| `automation/hooks.md` | procedure | Hooks: Event-Driven Scripts for Commands and Agent Lifecycle Events |
| `automation/standing-orders.md` | concept | Standing Orders: Permanent Operating Authority for Autonomous Agent Programs |
| `automation/tasks.md` | concept | Background Tasks: Activity Ledger for ACP Runs, Subagents, Cron, and CLI Operations |
| `channels/channel-routing.md` | concept | Channel Routing: Multi-Agent Message Routing, Binding Rules, and Session Isolation |
| `channels/channels-overview.md` | reference | Channels Overview: Built-In and Plugin Channels Supported by OpenClaw |
| `channels/discord-advanced.md` | reference | Discord Advanced: Role Routing, Forum Channels, Interactive Components, Voice, Troubleshooting |
| `channels/discord.md` | procedure | Discord Channel Setup: Bot Creation, Guild Config, Access Control, Slash Commands, and Voice |
| `channels/feishu.md` | procedure | Feishu (Lark) Channel: App Setup, Access Control, and Groups |
| `channels/googlechat.md` | procedure | Google Chat Channel: App Setup, Access Control, and Spaces |
| `channels/groups-and-broadcast.md` | concept | Group Messages and Broadcast Groups: requireMention, Group Routing, and Broadcast Targets |
| `channels/imessage-bluebubbles.md` | procedure | iMessage and BlueBubbles Channels: macOS Messages and Cross-Platform iMessage Setup |
| `channels/irc.md` | procedure | IRC Channel: Setup, Access Control, and Configuration |
| `channels/line.md` | procedure | Line Channel: Setup, Access Control, and Configuration |
| `channels/location.md` | reference | Location Sharing: Channel Location Message Handling and GPS Data |
| `channels/matrix-advanced.md` | reference | Matrix Advanced: Migration from Element, Push Rules, and Multi-Account Notes |
| `channels/matrix.md` | procedure | Matrix Channel: Homeserver Setup, E2E Encryption, Threads, Push Rules, and ACP Bindings |
| `channels/mattermost.md` | procedure | Mattermost Channel: Bot User Setup, Access Control, and Threads |
| `channels/msteams.md` | procedure | Microsoft Teams Channel: Azure Bot Setup, App Manifest, Federated Auth, and Local Dev |
| `channels/nextcloud-talk.md` | procedure | Nextcloud Talk Channel: Setup, Access Control, and Configuration |
| `channels/nostr.md` | procedure | Nostr Channel: Setup, Access Control, and Configuration |
| `channels/pairing.md` | concept | Channel Pairing: Linking Sessions to Channel Conversations |
| `channels/qa-channel.md` | reference | QA Channel: Automated Testing Channel for OpenClaw Integration Tests |
| `channels/qqbot.md` | procedure | QQ Bot Channel: Setup, Access Control, and Configuration |
| `channels/signal.md` | procedure | Signal Channel: signal-cli Setup, Linking Account, and Access Control |
| `channels/slack.md` | procedure | Slack Channel Setup: Socket Mode, App Token, Bot Token, Threading, and Exec Approvals |
| `channels/synology-chat.md` | procedure | Synology Chat Channel: Setup, Access Control, and Configuration |
| `channels/telegram.md` | procedure | Telegram Channel Setup: Bot Creation, Access Control, Groups, and Feature Reference |
| `channels/tlon.md` | procedure | Tlon Landscape Channel: Setup, Access Control, and Configuration |
| `channels/troubleshooting.md` | troubleshooting | Channel Troubleshooting: Connection Failures, Delivery Issues, and Common Errors |
| `channels/twitch.md` | procedure | Twitch Channel: Setup, Access Control, and Configuration |
| `channels/wechat.md` | procedure | WeChat Channel: Setup, Access Control, and Configuration |
| `channels/whatsapp.md` | procedure | WhatsApp Channel: QR Code Pairing, allowFrom Access Control, Groups, and Feature Reference |
| `channels/yuanbao.md` | procedure | Yuanbao Channel: Setup, Access Control, and Configuration |
| `channels/zalo.md` | procedure | Zalo Channel: Setup, Access Control, and Configuration |
| `channels/zalouser.md` | procedure | Zalo User Channel Plugin: Setup and Configuration |
| `cli/agent-cli.md` | reference | Agent CLI: openclaw agent and openclaw agents Commands |
| `cli/automation-cli.md` | reference | Automation CLI: cron, hooks, tasks, and wiki Commands |
| `cli/browser-sandbox-security-cli.md` | reference | CLI: Browser, Sandbox, Approvals, and Security Commands |
| `cli/clawbot-cmd.md` | reference | openclaw clawbot: Legacy Alias Namespace for QR Commands |
| `cli/cli-overview.md` | reference | OpenClaw CLI Overview: All Commands and Quick Reference |
| `cli/commitments-cmd.md` | reference | openclaw commitments: List and Dismiss Inferred Follow-Up Commitments |
| `cli/completion-cmd.md` | procedure | openclaw completion: Generate and Install Shell Completion Scripts |
| `cli/config-cli.md` | reference | Config CLI: openclaw config get, set, unset, validate, and show Commands |
| `cli/configure-cmd.md` | procedure | openclaw configure: Interactive Configuration Wizard |
| `cli/crestodian-cli.md` | reference | Crestodian CLI: Auth Profile and Credential Management |
| `cli/daemon-cmd.md` | reference | openclaw daemon: Legacy Alias for Gateway Service Management |
| `cli/directory-cmd.md` | reference | openclaw directory: Channel Directory Lookups |
| `cli/docs-cmd.md` | reference | openclaw docs: Search the Live Docs Index from the Terminal |
| `cli/gateway-cli.md` | reference | Gateway CLI: openclaw gateway start, stop, status, force, and Daemon Commands |
| `cli/infer-cli.md` | reference | Infer CLI: openclaw infer for One-Shot Model Inference and API Testing |
| `cli/mcp-cli.md` | reference | MCP CLI: Managing Model Context Protocol Servers with openclaw mcp |
| `cli/memory-cli.md` | reference | Memory CLI: openclaw memory search, list, and View Commands |
| `cli/migrate-cmd.md` | procedure | openclaw migrate: Import State from Another Agent System |
| `cli/misc-cli-commands.md` | reference | Miscellaneous CLI Commands: flows, tui, voicecall, doctor, dns, webhooks, and More |
| `cli/nodes-cmd.md` | reference | openclaw nodes: Manage Paired Nodes and Node Capabilities |
| `cli/nodes-pairing-message-cli.md` | reference | CLI: Nodes, Devices, Pairing, QR Code, and Message Commands |
| `cli/plugins-cli.md` | reference | Plugins CLI: openclaw plugins install, list, update, and info Commands |
| `cli/proxy-cmd.md` | reference | openclaw proxy: Local Debug Proxy and Traffic Capture Inspector |
| `cli/sessions-channels-models-secrets-cli.md` | reference | CLI: Sessions, Channels, Models, and Secrets Management Commands |
| `cli/setup-cmd.md` | procedure | openclaw setup: Initialize Config and Workspace |
| `cli/setup-health-misc-cli.md` | reference | CLI: dashboard, health, reset, completion, setup, onboard, and skills Commands |
| `cli/system-cmd.md` | reference | openclaw system: System Events, Heartbeat Control, and Presence |
| `cli/uninstall-cmd.md` | procedure | openclaw uninstall: Remove Gateway Service and Local Data |
| `cli/utility-cli.md` | reference | CLI Utilities: ACP, Status, Logs, Update, Backup, and Migrate Commands |
| `cli/voicecall-cmd.md` | reference | openclaw voicecall: Voice-Call Plugin CLI Commands |
| `concepts/agent-loop.md` | concept | Agent Loop: Execution Flow, Streaming, Tool Events, and Session Locking |
| `concepts/agent-overview.md` | concept | Agent Overview: Pi Agent, Sessions, Workspace, and Architecture |
| `concepts/agent-runtimes.md` | concept | Agent Runtime Modes: embedded-pi, Delegate Architecture, and CLI Backends |
| `concepts/agent-workspace.md` | concept | Agent Workspace: Directory Structure, Memory Files, and Bootstrap File Locations |
| `concepts/channel-docking.md` | concept | Channel Docking: Session-Channel Binding for Agent-Initiated Messages |
| `concepts/commitments.md` | concept | Commitments: Inferred Short-Lived Follow-Up Memories, Heartbeat Delivery |
| `concepts/compaction.md` | concept | Compaction: Auto-Summarization of Long Sessions, Token Limits, and Compaction Config |
| `concepts/context-engine.md` | concept | Context Engine: Prompt Assembly, Token Budget, Bootstrap Files, and System Prompt Construction |
| `concepts/date-time.md` | reference | Date and Time Handling: Timestamps in Messages, Prompts, and Tools |
| `concepts/delegate-architecture.md` | concept | Delegate Architecture: External Agent Processes and Runtime Routing |
| `concepts/dreaming.md` | concept | Dreaming: Background Memory Consolidation, DREAMS.md, and Dream Sweeps |
| `concepts/markdown-formatting.md` | concept | Markdown Formatting: Channel-Aware Rendering, Rich Output, and Message Formatting |
| `concepts/memory-overview.md` | concept | Memory Overview: MEMORY.md, Daily Notes, memory_search, and memory_get Tools |
| `concepts/messages.md` | concept | Message Handling: Inbound Routing, Reply Delivery, Attachments, and Message Format |
| `concepts/misc-concepts.md` | reference | Usage Tracking, Timezone Handling, and Experimental Features |
| `concepts/model-failover.md` | concept | Model Failover: Backup Model Chains, Auth Profiles, and Rate Limit Recovery |
| `concepts/models.md` | reference | Model Configuration: Provider/Model Format, Default Model, and Model Selection |
| `concepts/multi-agent.md` | concept | Multi-Agent Routing: Isolated Agents, Binding Rules, Workspace Isolation, Per-Agent Config |
| `concepts/network-overview.md` | concept | Network Architecture: Gateway Surfaces, Pairing, Discovery, and Security |
| `concepts/oauth-auth-profiles.md` | concept | OAuth and Auth Profiles: Multi-Credential Configuration, Provider Auth, and Profile Selection |
| `concepts/queue.md` | concept | Command Queue and Queue Steering: Serializing Agent Turns, Queue Modes, and Message Handling |
| `concepts/retry.md` | concept | Retry and Error Handling: Model Retries, Network Failures, and Compaction Retries |
| `concepts/sdk-overview.md` | concept | OpenClaw SDK Overview: TypeScript APIs for Plugins, Channels, and Providers |
| `concepts/session.md` | concept | Session Model: Persistent Conversations, Session Keys, Session Tool, and Pruning |
| `concepts/soul.md` | concept | Agent Soul: SOUL.md Personality File, Tone, and Character Configuration |
| `concepts/streaming.md` | concept | Response Streaming: Real-Time Assistant Deltas, Block Replies, and Channel Streaming Behavior |
| `concepts/system-prompt.md` | concept | System Prompt: Structure, Skill Prompts, Bootstrap Context, and Prompt Overlays |
| `concepts/typebox.md` | reference | TypeBox Schema Validation: Configuration Schemas, Plugin Manifest Validation |
| `concepts/typing-and-presence.md` | concept | Typing Indicators and Presence: Agent Processing Signals and Availability State |
| `gateway/authentication.md` | reference | Gateway Authentication: Token Auth, Trusted Proxy, and OAuth Auth Profiles |
| `gateway/bridge-protocol.md` | reference | Bridge Protocol: Historical TCP Bridge (Removed — Reference Only) |
| `gateway/cli-backends.md` | reference | CLI Backends: Local AI CLI Fallback with MCP Loopback Bridge |
| `gateway/config-agents-reference.md` | reference | Configuration Reference: agents.defaults, Workspace, Model, Heartbeat, and Per-Agent Overrides |
| `gateway/config-channels-reference.md` | reference | Channel Configuration Reference: allowFrom, botToken, groups, requireMention |
| `gateway/config-tools-reference.md` | reference | Tools Configuration Reference: Enabling Tools, MCP Servers, Elevated Exec, Web Search |
| `gateway/configuration-examples.md` | reference | Gateway Configuration Examples: Copy-Paste openclaw.json Configs |
| `gateway/configuration-overview.md` | procedure | Gateway Configuration Overview: openclaw.json Editing and Common Tasks |
| `gateway/gateway-features.md` | reference | Gateway Features: Background Process, Device Pairing, OpenShell, Gateway Lock |
| `gateway/gateway-runbook.md` | procedure | Gateway Service Runbook: Start, Status, Daemon, and Channel Probes |
| `gateway/health-diagnostics-logging.md` | procedure | Gateway Health, Diagnostics, and Logging: openclaw doctor, health, and logs |
| `gateway/heartbeat.md` | concept | Heartbeat: Periodic Agent Turns, HEARTBEAT.md, Active Hours, and heartbeat vs cron |
| `gateway/http-apis.md` | reference | Gateway HTTP APIs: OpenAI-Compatible Completions, OpenResponses, and Tools Invoke |
| `gateway/local-models.md` | procedure | Local AI Models: Ollama, LM Studio, vLLM, and SGLang Configuration |
| `gateway/network-discovery.md` | reference | Gateway Network Model, Multiple Gateways, mDNS Bonjour, and DNS-SD Discovery |
| `gateway/observability.md` | reference | Gateway Observability: OpenTelemetry Traces and Prometheus Metrics |
| `gateway/protocol.md` | reference | Gateway WebSocket RPC Protocol: Transport, Roles, Methods, and Device Pairing |
| `gateway/remote-access.md` | procedure | Remote Gateway Access: SSH Tunnel, Tailscale, HTTPS Proxy, and Remote CLI |
| `gateway/sandboxing.md` | concept | Sandboxing: Docker/Podman Tool Isolation, Elevated Exec, and Sandbox Modes |
| `gateway/secrets.md` | procedure | Gateway Secrets Management: SecretRef Syntax, Providers, and secrets apply |
| `gateway/security-audit.md` | reference | Security Audit: openclaw security audit Checks and Remediation |
| `gateway/security-overview.md` | reference | Gateway Security: Personal Assistant Model, Trust Boundaries, and Quick Hardening |
| `gateway/troubleshooting.md` | troubleshooting | Gateway Troubleshooting: Port Conflicts, Channel Failures, Model Errors, and Diagnostics |
| `getting-started/cli-automation.md` | procedure | CLI Automation: Scripted Non-Interactive Onboarding with openclaw onboard |
| `getting-started/docs-navigation.md` | reference | OpenClaw Documentation Map and Use Case Navigation |
| `getting-started/onboarding-overview.md` | concept | Onboarding Overview: Choosing Between CLI Wizard and macOS App Setup |
| `getting-started/onboarding-wizard.md` | procedure | OpenClaw Onboarding Wizard: openclaw onboard Interactive Setup |
| `getting-started/quickstart.md` | procedure | OpenClaw Quickstart: Install, Start Gateway, and Connect a Channel |
| `getting-started/what-is-openclaw.md` | concept | What Is OpenClaw: Self-Hosted AI Agent Gateway |
| `getting-started/workspace-bootstrap.md` | concept | Agent Workspace Bootstrap Files: AGENTS.md, SOUL.md, MEMORY.md Auto-Injection |
| `install/ansible.md` | procedure | Ansible: Automated OpenClaw Deployment with Ansible Playbooks |
| `install/cloud-platforms.md` | procedure | Cloud Platform Deployment: DigitalOcean, GCP, Azure, Fly.io, Oracle, and Hetzner |
| `install/development-install.md` | reference | Development Channels: Installing Pre-Release Builds and Dev Mode |
| `install/docker-vm-runtime.md` | reference | Docker VM Runtime: Shared Steps for Cloud VM OpenClaw Deployments |
| `install/docker.md` | procedure | Docker and Podman: Container Installation, docker-compose, ClawDock, and Volume Mounts |
| `install/hetzner.md` | procedure | Hetzner: Production VPS Deployment with Docker and Durable State |
| `install/hostinger.md` | procedure | Hostinger: 1-Click Managed OpenClaw or VPS Docker Install |
| `install/installation-methods.md` | procedure | Installation Methods: npm, One-Line Installer, Bun, Nix, and Node Requirements |
| `install/kubernetes.md` | procedure | Kubernetes: Deploying OpenClaw on Kubernetes |
| `install/migration.md` | procedure | Migration: Moving to OpenClaw from Claude, Hermes, and Other Platforms |
| `install/nix.md` | procedure | Nix: Declarative OpenClaw Install with Home Manager and nix-openclaw |
| `install/raspberry-pi.md` | procedure | Raspberry Pi Installation: ARM Setup and macOS VM |
| `install/updating.md` | procedure | Updating and Uninstalling OpenClaw: Update Methods and Clean Removal |
| `install/vps-hosting.md` | procedure | VPS Hosting: Running OpenClaw on Linux Servers and Cloud VPS Providers |
| `internals/ci-pipeline.md` | reference | CI Pipeline: Job Graph, Scope Gates, Runners, and Local Equivalents |
| `internals/design-plans.md` | reference | Design Plans: Codex Context Engine Harness and UI Channels Architecture |
| `internals/pi-integration.md` | reference | Pi Agent Integration: Architecture, Session Lifecycle, and Development Workflow |
| `internals/qa-automation.md` | reference | QA and E2E Automation: Test Framework, Test Matrix, and CI Integration |
| `internals/releasing.md` | reference | Release Process: Versioning, Release Checklist, and Publishing |
| `internals/superpowers-tweakcn-theme.md` | reference | Superpowers Spec: TweakCN Custom Theme Import Design |
| `memory/active-memory.md` | concept | Active Memory Plugin: Pre-Reply Memory Sub-Agent for Proactive Context Injection |
| `memory/memory-config-reference.md` | reference | Memory Configuration Reference: memorySearch, QMD, Active Memory, and Embedding Settings |
| `memory/memory-honcho.md` | procedure | Honcho Memory Backend: Cloud-Based Persistent Memory for OpenClaw Agents |
| `memory/memory-lancedb.md` | procedure | LanceDB Memory Plugin: Vector Database Backend for High-Scale Memory Search |
| `memory/memory-qmd.md` | procedure | QMD Memory Engine: Local Search Sidecar with Reranking, Query Expansion, and Extra Index Paths |
| `memory/memory-search.md` | reference | Memory Search: Hybrid BM25 + Vector Embeddings, Embedding Providers, and memory_search Tool |
| `memory/memory-wiki.md` | procedure | Memory Wiki Plugin: Structured Wiki Vault from Agent Memory with Claims and Evidence |
| `nodes/images.md` | reference | Image and Media Support: Send, Gateway, and Agent Reply Pipeline |
| `nodes/location-command.md` | reference | Location Command: node.invoke location.get, Permissions, and Android Foreground |
| `nodes/media-understanding.md` | concept | Media Understanding: Image, Video, and Audio Processing from Node Cameras and Microphones |
| `nodes/nodes-overview.md` | concept | Nodes Overview: iOS/Android Hardware Nodes for Camera, Audio, Voice, and Location |
| `nodes/troubleshooting.md` | troubleshooting | Node Troubleshooting: Pairing, Permissions, Foreground Requirements, and Error Codes |
| `nodes/voicewake.md` | reference | Voice Wake: Global Wake Words Owned by the Gateway, Sync, and Routing |
| `platforms/digitalocean.md` | procedure | DigitalOcean: Running OpenClaw on a Droplet ($6/month) |
| `platforms/ios-android.md` | procedure | iOS and Android Nodes: Mobile App Setup, Pairing, and Node Features |
| `platforms/macos-app-features.md` | reference | macOS App Features: Menu Bar Icon, Logging, Remote Control, Skills UI, and Voice Wake |
| `platforms/macos-app.md` | procedure | macOS App: Menu Bar, Permissions, Bundled Gateway, and macOS-Specific Features |
| `platforms/macos-features.md` | reference | macOS Features: Canvas, Peekaboo Screen Capture, Voice Overlay, and Voice Wake |
| `platforms/macos-gateway-lifecycle.md` | reference | macOS Gateway Lifecycle: launchd, Health Checks, and WebChat Embedding |
| `platforms/macos-internals.md` | reference | macOS Internals: Dev Setup, Code Signing, XPC, Child Process, and Logging |
| `platforms/oracle.md` | procedure | Oracle Cloud: Running OpenClaw on Always Free ARM (24GB RAM, $0/month) |
| `platforms/other-platforms.md` | reference | Platform Guides: Linux, Windows, Raspberry Pi |
| `platforms/platforms-overview.md` | concept | Platforms Overview: OS Support, VPS Hosting, and Gateway Service Install |
| `plugins/agent-tools.md` | reference | Plugin Agent Tools: Registering and Providing Tools to the Agent |
| `plugins/architecture-internals-2.md` | reference | Plugin Internals: Message Schemas, Context Engine Plugins, and Capability Checklist |
| `plugins/architecture-internals.md` | reference | Plugin Architecture Internals: Load Pipeline, Registry, Provider Hooks, and HTTP Routes |
| `plugins/building-plugins.md` | procedure | Building Plugins: Step-by-Step Guide to Creating OpenClaw Plugin Packages |
| `plugins/bundles.md` | procedure | Plugin Bundles: Installing Curated Plugin Groups |
| `plugins/codex-computer-use.md` | reference | Codex Computer Use Plugin: Desktop Control via Computer Use APIs |
| `plugins/codex-harness.md` | reference | Codex Harness Plugin: OpenClaw Integration with OpenAI Codex Agent Runtime |
| `plugins/community.md` | reference | Community Plugins: Third-Party Plugin Directory and Contributing |
| `plugins/compatibility.md` | reference | Plugin Compatibility: SDK Version Support Matrix and Version Requirements |
| `plugins/google-meet.md` | procedure | Google Meet Plugin: Joining Calls, Transcription, and Meeting Participation |
| `plugins/hooks.md` | reference | Plugin Hooks: Lifecycle Callbacks for Agent Events, Tool Calls, and Message Handling |
| `plugins/message-presentation.md` | reference | Message Presentation Plugin: Control UI Display and Rich Message Rendering |
| `plugins/openprose.md` | procedure | OpenProse Plugin: .prose Workflow Files and Slash Commands |
| `plugins/plugin-architecture.md` | concept | Plugin Architecture: Types, Installation, and Configuration |
| `plugins/plugin-manifest-2.md` | reference | Plugin Manifest: Model Support, Model Catalog, Provider Endpoints, and Pricing |
| `plugins/plugin-manifest.md` | reference | Plugin Manifest: Declaring Channels, Providers, Tools, and All Manifest Fields |
| `plugins/sdk-agent-harness.md` | reference | SDK Agent Harness: Building Agent Runtime Adapters and Harness Plugins |
| `plugins/sdk-channel-plugins.md` | procedure | SDK Channel Plugins: Building Custom Messaging Channel Integrations |
| `plugins/sdk-channel-turn.md` | reference | SDK Channel Turn: Inbound Message Handling, Turn Context, and Reply Delivery |
| `plugins/sdk-entrypoints.md` | reference | Plugin Entrypoints: Registration with the Gateway at Load Time |
| `plugins/sdk-migration-2.md` | reference | Plugin SDK Migration: Active Deprecations, Removal Timeline, and Warning Suppression |
| `plugins/sdk-migration.md` | procedure | Plugin SDK Migration Guide: Breaking Changes, Import Paths, and Deprecation Timeline |
| `plugins/sdk-overview.md` | concept | Plugin SDK Overview: TypeScript APIs for Building Channels, Providers, and Tools |
| `plugins/sdk-provider-plugins.md` | procedure | SDK Provider Plugins: Building Custom AI Model Provider Integrations |
| `plugins/sdk-runtime.md` | reference | SDK Runtime APIs: Config Access, Logging, and Gateway Interaction from Plugin Code |
| `plugins/sdk-setup.md` | procedure | Plugin SDK Setup: Project Structure, package.json, TypeScript Config, and Dev Workflow |
| `plugins/sdk-subpaths.md` | reference | SDK Import Subpaths: Complete Map of @openclaw/sdk Imports and APIs |
| `plugins/sdk-testing.md` | procedure | Plugin SDK Testing: Unit Tests, Integration Tests, and QA Runner Setup |
| `plugins/skill-workshop.md` | procedure | Skill Workshop Plugin: Creating, Testing, and Publishing Agent Skills |
| `plugins/voice-call.md` | procedure | Voice Call Plugin: Real-Time Voice Conversations with OpenClaw Agents |
| `plugins/webhooks.md` | procedure | Webhooks Plugin: Inbound Webhook Endpoints for Triggering Agent Runs |
| `plugins/zalouser.md` | procedure | Zalo User Plugin: Setup and Configuration |
| `providers/alibaba.md` | procedure | Alibaba Cloud Provider: Setup, Configuration, and Model Reference |
| `providers/anthropic.md` | procedure | Anthropic Claude Provider: Claude Opus, Sonnet, Haiku, OAuth, and Prompt Caching |
| `providers/arcee.md` | procedure | Cohere Provider: Setup, Configuration, and Model Reference |
| `providers/azure-speech.md` | procedure | Azure Speech Provider: Setup, Configuration, and Model Reference |
| `providers/bedrock.md` | procedure | AWS Bedrock Provider: Claude on Bedrock, Mantle Multi-Region, and IAM Setup |
| `providers/cerebras.md` | procedure | Cerebras Provider: Setup, Configuration, and Model Reference |
| `providers/chutes.md` | procedure | Chutes Provider: Setup, Configuration, and Model Reference |
| `providers/claude-max-proxy.md` | procedure | Claude Max Proxy Provider: Setup, Configuration, and Model Reference |
| `providers/cloudflare.md` | procedure | Cloudflare AI Gateway Provider: Setup, Configuration, and Model Reference |
| `providers/comfy.md` | procedure | ComfyUI Provider: Setup, Configuration, and Model Reference |
| `providers/deepgram.md` | procedure | Deepgram STT Provider: Setup, Configuration, and Model Reference |
| `providers/deepinfra.md` | procedure | DeepInfra Provider: Setup, Configuration, and Model Reference |
| `providers/deepseek.md` | procedure | DeepSeek Provider: Setup, Configuration, and Model Reference |
| `providers/elevenlabs.md` | procedure | ElevenLabs TTS Provider: Setup, Configuration, and Model Reference |
| `providers/fal.md` | procedure | FAL AI Provider: Setup, Configuration, and Model Reference |
| `providers/fireworks.md` | procedure | Fireworks AI Provider: Setup, Configuration, and Model Reference |
| `providers/github-copilot.md` | procedure | GitHub Copilot Provider: Setup, Configuration, and Model Reference |
| `providers/glm.md` | procedure | GLM Z.AI Provider: Setup, Configuration, and Model Reference |
| `providers/google.md` | procedure | Google Gemini Provider: Gemini Pro, Flash, Vertex AI, and Grounding Setup |
| `providers/gradium.md` | procedure | Gradium Provider: Setup, Configuration, and Model Reference |
| `providers/groq.md` | procedure | Groq Provider: Setup, Configuration, and Model Reference |
| `providers/huggingface.md` | procedure | HuggingFace Provider: Setup, Configuration, and Model Reference |
| `providers/inferrs.md` | procedure | Inferrs Provider: Setup, Configuration, and Model Reference |
| `providers/inworld.md` | procedure | InWorld Provider: Setup, Configuration, and Model Reference |
| `providers/kilocode.md` | procedure | Kilocode Provider: Setup, Configuration, and Model Reference |
| `providers/litellm.md` | procedure | LiteLLM Provider: Setup, Configuration, and Model Reference |
| `providers/lmstudio.md` | procedure | LM Studio Provider: Setup, Configuration, and Model Reference |
| `providers/minimax.md` | procedure | MiniMax Provider: Setup, Configuration, and Model Reference |
| `providers/mistral.md` | procedure | Mistral Provider: Setup, Configuration, and Model Reference |
| `providers/models-reference.md` | procedure | Models Reference Provider: Setup, Configuration, and Model Reference |
| `providers/moonshot.md` | procedure | Moonshot Kimi Provider: Setup, Configuration, and Model Reference |
| `providers/nvidia.md` | procedure | NVIDIA NIM Provider: Setup, Configuration, and Model Reference |
| `providers/ollama-advanced.md` | reference | Ollama Advanced: Cloud Models, Vision, Web Search, and Troubleshooting |
| `providers/ollama.md` | procedure | Ollama Provider: Local LLM Setup, Auto-Discovery, Vision, and Web Search |
| `providers/openai-advanced.md` | reference | OpenAI Advanced: Azure OpenAI Endpoints, Codex OAuth, and Voice Configuration |
| `providers/openai.md` | procedure | OpenAI Provider: GPT-4o, Image Generation, Voice, Codex, and Azure OpenAI Setup |
| `providers/opencode-go.md` | procedure | OpenCode Go Provider: Setup, Configuration, and Model Reference |
| `providers/opencode.md` | procedure | OpenCode Provider: Setup, Configuration, and Model Reference |
| `providers/openrouter.md` | procedure | OpenRouter Provider: Setup, Configuration, and Model Reference |
| `providers/perplexity-provider.md` | procedure | Perplexity Provider: Setup, Configuration, and Model Reference |
| `providers/providers-overview.md` | reference | AI Providers Overview: All Supported Providers by Category |
| `providers/qianfan.md` | procedure | Baidu Qianfan Provider: Setup, Configuration, and Model Reference |
| `providers/qwen.md` | procedure | Alibaba Qwen Provider: Setup, Configuration, and Model Reference |
| `providers/runway.md` | procedure | Runway Provider: Setup, Configuration, and Model Reference |
| `providers/senseaudio.md` | procedure | SenseAudio Provider: Setup, Configuration, and Model Reference |
| `providers/sglang.md` | procedure | SGLang Provider: Setup, Configuration, and Model Reference |
| `providers/stepfun.md` | procedure | Stepfun Provider: Setup, Configuration, and Model Reference |
| `providers/synthetic.md` | procedure | Synthetic Provider: Setup, Configuration, and Model Reference |
| `providers/tencent.md` | procedure | Tencent Provider: Setup, Configuration, and Model Reference |
| `providers/together.md` | procedure | Together AI Provider: Setup, Configuration, and Model Reference |
| `providers/venice.md` | procedure | Venice.ai Provider: Setup, Configuration, and Model Reference |
| `providers/vercel-ai-gateway.md` | procedure | Vercel AI Gateway Provider: Setup, Configuration, and Model Reference |
| `providers/vllm.md` | procedure | vLLM Provider: Setup, Configuration, and Model Reference |
| `providers/volcengine.md` | procedure | Volcengine Provider: Setup, Configuration, and Model Reference |
| `providers/vydra.md` | procedure | Vydra Provider: Setup, Configuration, and Model Reference |
| `providers/xai.md` | procedure | XAI Grok Provider: Setup, Configuration, and Model Reference |
| `providers/xiaomi.md` | procedure | Xiaomi Provider: Setup, Configuration, and Model Reference |
| `providers/zai.md` | integration | Z.AI: GLM Model Provider Setup and Configuration |
| `reference/agents-default.md` | reference | AGENTS.md Default Content: Default Agent Instructions for OpenClaw |
| `reference/api-usage-costs.md` | reference | API Usage Costs: Cost Tracking, Reporting, and Provider Pricing Reference |
| `reference/auth-credential-semantics.md` | reference | Auth Credential Semantics: Profile Resolution, Eligibility, and Routing Rules |
| `reference/credits.md` | reference | Credits, Contributors, and License |
| `reference/device-models.md` | reference | Device Models Reference: Node Hardware Model Identifiers |
| `reference/docs-authoring-guide.md` | reference | Docs Authoring Guide: Mintlify Link Rules, i18n Policy, and Documentation Structure |
| `reference/memory-config-quick.md` | reference | Memory Configuration Quick Reference: All memorySearch and QMD Fields |
| `reference/modernization-plan.md` | reference | Application Modernization Plan: Roadmap and Architectural Evolution |
| `reference/prompt-caching.md` | reference | Prompt Caching: Reducing API Costs by Caching Prompt Prefixes |
| `reference/protocols.md` | reference | Protocols Reference: Rich Output Protocol and RPC |
| `reference/sdk-api-design.md` | reference | SDK API Design: Design Principles, Conventions, and Architecture Decisions |
| `reference/secretref-surfaces.md` | reference | SecretRef Credential Surfaces: Which Configuration Fields Support Secret References |
| `reference/session-management.md` | reference | Session Management and Compaction Reference: Internals and Configuration |
| `reference/testing.md` | reference | Testing Reference: Test Suite Types, Configuration, and Running Tests |
| `reference/token-use.md` | reference | Token Usage: Counting, Tracking, and Optimizing Token Consumption |
| `reference/transcript-hygiene.md` | reference | Transcript Hygiene: Managing Session History, Sensitive Data, and File Format |
| `reference/wizard-reference.md` | reference | Wizard Configuration Reference: All Onboarding Wizard Fields and Automation Options |
| `reference/workspace-templates-dev.md` | reference | Workspace Dev Templates: AGENTS.dev.md, SOUL.dev.md, TOOLS.dev.md, and IDENTITY.dev.md |
| `reference/workspace-templates.md` | reference | Workspace Template Files: Default AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, and More |
| `security/contributing.md` | reference | Contributing to the Threat Model: Format, Structure, and Submission Process |
| `security/formal-verification.md` | reference | Formal Verification: Security-Critical Code Verification in OpenClaw |
| `security/network-proxy.md` | procedure | Network Proxy Security: nginx, Caddy, TLS Termination, and Reverse Proxy Config |
| `security/threat-model-2.md` | reference | Threat Model Part 2: Detailed Threat Categories and Countermeasures |
| `security/threat-model.md` | reference | OpenClaw Threat Model: Attack Vectors, Trust Boundaries, and Security Mitigations |
| `tools/acp-agents-setup.md` | procedure | ACP Agents Setup: Configuration, Session Binding, and Delivery Model |
| `tools/acp-agents.md` | procedure | ACP Agents: Running Claude Code and External Coding Agents via ACP Protocol |
| `tools/agent-communication.md` | reference | Agent Communication Tools: agent_send, Channel Messaging, and Reactions |
| `tools/brave-search-setup.md` | procedure | Brave Search Setup: API Key, Plan Details, and web_search Configuration |
| `tools/browser-advanced.md` | reference | Browser Advanced: Remote CDP, Browserless, Login Persistence, WSL2, and Linux Troubleshooting |
| `tools/browser.md` | procedure | Browser Tool: Chromium Control via CDP, Navigation, Forms, and Screenshots |
| `tools/clawhub-lobster.md` | reference | ClawHub and Lobster: Tool Discovery and Multi-Agent Sandbox Tool Framework |
| `tools/duckduckgo-search.md` | integration | DuckDuckGo Web Search: Key-Free Fallback Search Provider (Experimental) |
| `tools/exec-approvals.md` | procedure | Exec Approvals: Human Confirmation Gates for Shell Command Execution |
| `tools/exec.md` | procedure | Exec Tool: Shell Command Execution, Elevated Mode, and Code Execution |
| `tools/file-patching.md` | procedure | File Patching: apply_patch Tool and Diffs for Code Modification |
| `tools/firecrawl.md` | integration | Firecrawl: Search, Scrape, and web_fetch Fallback with Bot Circumvention |
| `tools/image-generation.md` | procedure | Image Generation Tool: DALL-E, fal.ai, ComfyUI Providers and Configuration |
| `tools/kimi-search.md` | integration | Kimi Web Search: AI-Synthesized Answers via Moonshot Web Search |
| `tools/llm-task.md` | reference | LLM Task Tool: Discrete LLM Calls for Classification, Extraction, and Summarization |
| `tools/media-overview.md` | reference | Media Tools Overview: Image, Video, Audio Generation and Understanding |
| `tools/minimax-search.md` | integration | MiniMax Web Search: Coding Plan Search API Integration |
| `tools/misc-tools.md` | reference | Miscellaneous Tools: btw, Capability Cookbook, Loop Detection, Trajectory, TokenJuice |
| `tools/music-generation.md` | procedure | Music Generation Tool: Audio Generation and Provider Configuration |
| `tools/ollama-search.md` | integration | Ollama Web Search: Local or Hosted Search via Ollama API |
| `tools/pdf.md` | procedure | PDF Tool: Reading and Extracting Text from PDF Files |
| `tools/perplexity-setup.md` | procedure | Perplexity Search Setup: API Key, Sonar Model, and OpenRouter Compatibility |
| `tools/plugin-tool.md` | concept | Plugin Tool: Agent Interaction with Plugin-Provided Tools and Capabilities |
| `tools/skills.md` | procedure | Skills: Contextual Instruction Files, Loading Skills, and Creating Custom Skills |
| `tools/slash-commands-advanced.md` | reference | Slash Commands Advanced: Plugin Commands, Custom Commands, and Channel Behavior |
| `tools/slash-commands.md` | reference | Slash Commands: /new, /reset, /stop, /help, and All Built-In Commands |
| `tools/subagents.md` | concept | Subagents: Spawning and Coordinating Child Agents for Parallel Work |
| `tools/tavily.md` | integration | Tavily: Search and Extract Tools for AI Applications |
| `tools/thinking.md` | procedure | Thinking Tool: Extended Reasoning, Thinking Levels, and Reasoning Configuration |
| `tools/tokenjuice.md` | concept | Tokenjuice: Compact Noisy Exec and Bash Tool Results |
| `tools/tools-overview.md` | reference | Agent Tools Overview: Exec, File, Search, Browser, Media, and Communication Tools |
| `tools/tts-advanced.md` | reference | TTS Advanced: Personas, Slash Commands, Per-User Preferences, and Output Formats |
| `tools/tts.md` | procedure | Text-to-Speech Tool: TTS Providers, Personas, Voice Config, and Auto-TTS |
| `tools/video-generation.md` | procedure | Video Generation Tool: Runway and Provider Configuration |
| `tools/web-search-additional.md` | reference | Additional Search Providers: Gemini, Grok, Kimi, Ollama, SearXNG, MiniMax, and Web Fetch |
| `tools/web-search-tools.md` | reference | Web Search Tools: Perplexity, Brave, Exa, DuckDuckGo, Tavily, Firecrawl, and More |
| `troubleshooting/browser-wsl2-cdp.md` | troubleshooting | WSL2 + Windows + Remote Chrome CDP Troubleshooting |
| `troubleshooting/debug-and-flags.md` | troubleshooting | Node Debug and Diagnostic Flags: Debugging Node Issues and Gateway Diagnostics |
| `troubleshooting/debugging.md` | procedure | Debugging Guide: Log Levels, Trace Mode, Diagnostic Commands, and Issue Reporting |
| `troubleshooting/environment-variables.md` | reference | Environment Variables Reference: All OpenClaw Env Vars and Their Effects |
| `troubleshooting/faq-2.md` | reference | FAQ Part 2: Sessions, Remote Gateways, Security, and Media Questions |
| `troubleshooting/faq-first-run.md` | troubleshooting | First Run FAQ: Common Setup Issues, API Keys, and Initial Configuration Problems |
| `troubleshooting/faq-models.md` | troubleshooting | Models FAQ: Choosing Models, API Keys, Rate Limits, and Model-Specific Issues |
| `troubleshooting/faq.md` | reference | FAQ: Common Questions About OpenClaw Setup, Config, Models, and Channels |
| `troubleshooting/general-troubleshooting.md` | troubleshooting | General Troubleshooting: Entry Points for Gateway, Channel, Model, and Tool Issues |
| `troubleshooting/gpt55-codex-parity.md` | reference | GPT-5 and Codex Agentic Parity: Compatibility Notes and Maintainer Guidance |
| `troubleshooting/logging-guide.md` | procedure | Logging Guide: File Logs, Console Output, Log Levels, and CLI Tailing |
| `troubleshooting/scripts-and-help.md` | reference | Help Index and Scripts: Helper Scripts and Utility Reference |
| `troubleshooting/testing-live.md` | procedure | Live Testing: Testing with Real Providers, Models, and Channels |
| `troubleshooting/testing.md` | procedure | Testing Guide: Running Unit, Integration, E2E, and Live Tests |
| `web-ui/control-ui.md` | procedure | Control UI: Browser Dashboard for Chat, Config, Sessions, and Logs |
| `web-ui/webchat-dashboard-tui.md` | reference | WebChat, Dashboard, and TUI: Web and Terminal Interfaces for OpenClaw |

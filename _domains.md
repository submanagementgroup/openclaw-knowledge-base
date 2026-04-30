---
domain: _meta
topic: "Domain Overview"
type: reference
keywords:
  - domains
  - knowledge base domains
  - domain guide
source: auto-generated
---

# OpenClaw Knowledge Base Domains

## `automation/` — Cron jobs, heartbeat, hooks, tasks, webhooks, and standing orders

6 files.

- `automation/automation-misc.md` — Automation: Auth Monitoring, ClawFlow, Polling, Webhooks, Gmail PubSub, and Troubleshooting
- `automation/automation-overview.md` — Automation Overview: Cron, Heartbeat, Hooks, Webhooks, Tasks, and Standing Orders
- `automation/cron-jobs.md` — Cron Jobs: Scheduling Periodic Agent Runs, One-Shot Jobs, and Webhook Triggers
- `automation/hooks.md` — Hooks: Event-Driven Scripts for Commands and Agent Lifecycle Events
- `automation/standing-orders.md` — Standing Orders: Permanent Operating Authority for Autonomous Agent Programs
- `automation/tasks.md` — Background Tasks: Activity Ledger for ACP Runs, Subagents, Cron, and CLI Operations

## `channels/` — Messaging channel setup: Telegram, WhatsApp, Discord, Slack, Matrix, Teams, and 20+ more

32 files.

- `channels/channel-routing.md` — Channel Routing: Multi-Agent Message Routing, Binding Rules, and Session Isolation
- `channels/channels-overview.md` — Channels Overview: Built-In and Plugin Channels Supported by OpenClaw
- `channels/discord-advanced.md` — Discord Advanced: Role Routing, Forum Channels, Interactive Components, Voice, Troubleshooting
- `channels/discord.md` — Discord Channel Setup: Bot Creation, Guild Config, Access Control, Slash Commands, and Voice
- `channels/feishu.md` — Feishu (Lark) Channel: App Setup, Access Control, and Groups
- `channels/googlechat.md` — Google Chat Channel: App Setup, Access Control, and Spaces
- `channels/groups-and-broadcast.md` — Group Messages and Broadcast Groups: requireMention, Group Routing, and Broadcast Targets
- `channels/imessage-bluebubbles.md` — iMessage and BlueBubbles Channels: macOS Messages and Cross-Platform iMessage Setup
- `channels/irc.md` — IRC Channel: Setup, Access Control, and Configuration
- `channels/line.md` — Line Channel: Setup, Access Control, and Configuration
- `channels/location.md` — Location Sharing: Channel Location Message Handling and GPS Data
- `channels/matrix-advanced.md` — Matrix Advanced: Migration from Element, Push Rules, and Multi-Account Notes
- `channels/matrix.md` — Matrix Channel: Homeserver Setup, E2E Encryption, Threads, Push Rules, and ACP Bindings
- `channels/mattermost.md` — Mattermost Channel: Bot User Setup, Access Control, and Threads
- `channels/msteams.md` — Microsoft Teams Channel: Azure Bot Setup, App Manifest, Federated Auth, and Local Dev
- `channels/nextcloud-talk.md` — Nextcloud Talk Channel: Setup, Access Control, and Configuration
- `channels/nostr.md` — Nostr Channel: Setup, Access Control, and Configuration
- `channels/pairing.md` — Channel Pairing: Linking Sessions to Channel Conversations
- `channels/qa-channel.md` — QA Channel: Automated Testing Channel for OpenClaw Integration Tests
- `channels/qqbot.md` — QQ Bot Channel: Setup, Access Control, and Configuration
- ... and 12 more

## `cli/` — CLI command reference for all openclaw subcommands

16 files.

- `cli/agent-cli.md` — Agent CLI: openclaw agent and openclaw agents Commands
- `cli/automation-cli.md` — Automation CLI: cron, hooks, tasks, and wiki Commands
- `cli/browser-sandbox-security-cli.md` — CLI: Browser, Sandbox, Approvals, and Security Commands
- `cli/cli-overview.md` — OpenClaw CLI Overview: All Commands and Quick Reference
- `cli/config-cli.md` — Config CLI: openclaw config get, set, unset, validate, and show Commands
- `cli/crestodian-cli.md` — Crestodian CLI: Auth Profile and Credential Management
- `cli/gateway-cli.md` — Gateway CLI: openclaw gateway start, stop, status, force, and Daemon Commands
- `cli/infer-cli.md` — Infer CLI: openclaw infer for One-Shot Model Inference and API Testing
- `cli/mcp-cli.md` — MCP CLI: Managing Model Context Protocol Servers with openclaw mcp
- `cli/memory-cli.md` — Memory CLI: openclaw memory search, list, and View Commands
- `cli/misc-cli-commands.md` — Miscellaneous CLI Commands: flows, tui, voicecall, doctor, dns, webhooks, and More
- `cli/nodes-pairing-message-cli.md` — CLI: Nodes, Devices, Pairing, QR Code, and Message Commands
- `cli/plugins-cli.md` — Plugins CLI: openclaw plugins install, list, update, and info Commands
- `cli/sessions-channels-models-secrets-cli.md` — CLI: Sessions, Channels, Models, and Secrets Management Commands
- `cli/setup-health-misc-cli.md` — CLI: dashboard, health, reset, completion, setup, onboard, and skills Commands
- `cli/utility-cli.md` — CLI Utilities: ACP, Status, Logs, Update, Backup, and Migrate Commands

## `concepts/` — Core concepts: agent loop, memory, sessions, models, context, multi-agent routing

29 files.

- `concepts/agent-loop.md` — Agent Loop: Execution Flow, Streaming, Tool Events, and Session Locking
- `concepts/agent-overview.md` — Agent Overview: Pi Agent, Sessions, Workspace, and Architecture
- `concepts/agent-runtimes.md` — Agent Runtime Modes: embedded-pi, Delegate Architecture, and CLI Backends
- `concepts/agent-workspace.md` — Agent Workspace: Directory Structure, Memory Files, and Bootstrap File Locations
- `concepts/channel-docking.md` — Channel Docking: Session-Channel Binding for Agent-Initiated Messages
- `concepts/commitments.md` — Commitments: Inferred Short-Lived Follow-Up Memories, Heartbeat Delivery
- `concepts/compaction.md` — Compaction: Auto-Summarization of Long Sessions, Token Limits, and Compaction Config
- `concepts/context-engine.md` — Context Engine: Prompt Assembly, Token Budget, Bootstrap Files, and System Prompt Construction
- `concepts/date-time.md` — Date and Time Handling: Timestamps in Messages, Prompts, and Tools
- `concepts/delegate-architecture.md` — Delegate Architecture: External Agent Processes and Runtime Routing
- `concepts/dreaming.md` — Dreaming: Background Memory Consolidation, DREAMS.md, and Dream Sweeps
- `concepts/markdown-formatting.md` — Markdown Formatting: Channel-Aware Rendering, Rich Output, and Message Formatting
- `concepts/memory-overview.md` — Memory Overview: MEMORY.md, Daily Notes, memory_search, and memory_get Tools
- `concepts/messages.md` — Message Handling: Inbound Routing, Reply Delivery, Attachments, and Message Format
- `concepts/misc-concepts.md` — Usage Tracking, Timezone Handling, and Experimental Features
- `concepts/model-failover.md` — Model Failover: Backup Model Chains, Auth Profiles, and Rate Limit Recovery
- `concepts/models.md` — Model Configuration: Provider/Model Format, Default Model, and Model Selection
- `concepts/multi-agent.md` — Multi-Agent Routing: Isolated Agents, Binding Rules, Workspace Isolation, Per-Agent Config
- `concepts/network-overview.md` — Network Architecture: Gateway Surfaces, Pairing, Discovery, and Security
- `concepts/oauth-auth-profiles.md` — OAuth and Auth Profiles: Multi-Credential Configuration, Provider Auth, and Profile Selection
- ... and 9 more

## `gateway/` — Gateway service: configuration, security, secrets, sandboxing, protocols, and APIs

21 files.

- `gateway/authentication.md` — Gateway Authentication: Token Auth, Trusted Proxy, and OAuth Auth Profiles
- `gateway/config-agents-reference.md` — Configuration Reference: agents.defaults, Workspace, Model, Heartbeat, and Per-Agent Overrides
- `gateway/config-channels-reference.md` — Channel Configuration Reference: allowFrom, botToken, groups, requireMention
- `gateway/config-tools-reference.md` — Tools Configuration Reference: Enabling Tools, MCP Servers, Elevated Exec, Web Search
- `gateway/configuration-examples.md` — Gateway Configuration Examples: Copy-Paste openclaw.json Configs
- `gateway/configuration-overview.md` — Gateway Configuration Overview: openclaw.json Editing and Common Tasks
- `gateway/gateway-features.md` — Gateway Features: Background Process, Device Pairing, OpenShell, Gateway Lock
- `gateway/gateway-runbook.md` — Gateway Service Runbook: Start, Status, Daemon, and Channel Probes
- `gateway/health-diagnostics-logging.md` — Gateway Health, Diagnostics, and Logging: openclaw doctor, health, and logs
- `gateway/heartbeat.md` — Heartbeat: Periodic Agent Turns, HEARTBEAT.md, Active Hours, and heartbeat vs cron
- `gateway/http-apis.md` — Gateway HTTP APIs: OpenAI-Compatible Completions, OpenResponses, and Tools Invoke
- `gateway/local-models.md` — Local AI Models: Ollama, LM Studio, vLLM, and SGLang Configuration
- `gateway/network-discovery.md` — Gateway Network Model, Multiple Gateways, mDNS Bonjour, and DNS-SD Discovery
- `gateway/observability.md` — Gateway Observability: OpenTelemetry Traces and Prometheus Metrics
- `gateway/protocol.md` — Gateway WebSocket RPC Protocol: Transport, Roles, Methods, and Device Pairing
- `gateway/remote-access.md` — Remote Gateway Access: SSH Tunnel, Tailscale, HTTPS Proxy, and Remote CLI
- `gateway/sandboxing.md` — Sandboxing: Docker/Podman Tool Isolation, Elevated Exec, and Sandbox Modes
- `gateway/secrets.md` — Gateway Secrets Management: SecretRef Syntax, Providers, and secrets apply
- `gateway/security-audit.md` — Security Audit: openclaw security audit Checks and Remediation
- `gateway/security-overview.md` — Gateway Security: Personal Assistant Model, Trust Boundaries, and Quick Hardening
- ... and 1 more

## `getting-started/` — Quickstart, installation, onboarding wizard, and workspace bootstrap

5 files.

- `getting-started/docs-navigation.md` — OpenClaw Documentation Map and Use Case Navigation
- `getting-started/onboarding-wizard.md` — OpenClaw Onboarding Wizard: openclaw onboard Interactive Setup
- `getting-started/quickstart.md` — OpenClaw Quickstart: Install, Start Gateway, and Connect a Channel
- `getting-started/what-is-openclaw.md` — What Is OpenClaw: Self-Hosted AI Agent Gateway
- `getting-started/workspace-bootstrap.md` — Agent Workspace Bootstrap Files: AGENTS.md, SOUL.md, MEMORY.md Auto-Injection

## `install/` — Platform-specific installation: Docker, Kubernetes, cloud VPS, and package managers

10 files.

- `install/ansible.md` — Ansible: Automated OpenClaw Deployment with Ansible Playbooks
- `install/cloud-platforms.md` — Cloud Platform Deployment: DigitalOcean, GCP, Azure, Fly.io, Oracle, and Hetzner
- `install/development-install.md` — Development Channels: Installing Pre-Release Builds and Dev Mode
- `install/docker.md` — Docker and Podman: Container Installation, docker-compose, ClawDock, and Volume Mounts
- `install/installation-methods.md` — Installation Methods: npm, One-Line Installer, Bun, Nix, and Node Requirements
- `install/kubernetes.md` — Kubernetes: Deploying OpenClaw on Kubernetes
- `install/migration.md` — Migration: Moving to OpenClaw from Claude, Hermes, and Other Platforms
- `install/raspberry-pi.md` — Raspberry Pi Installation: ARM Setup and macOS VM
- `install/updating.md` — Updating and Uninstalling OpenClaw: Update Methods and Clean Removal
- `install/vps-hosting.md` — VPS Hosting: Running OpenClaw on Linux Servers and Cloud VPS Providers

## `internals/` — Internals: Pi integration, CI pipeline, QA automation, design plans, and release process

6 files.

- `internals/ci-pipeline.md` — CI Pipeline: Job Graph, Scope Gates, Runners, and Local Equivalents
- `internals/design-plans.md` — Design Plans: Codex Context Engine Harness and UI Channels Architecture
- `internals/pi-integration.md` — Pi Agent Integration: Architecture, Session Lifecycle, and Development Workflow
- `internals/qa-automation.md` — QA and E2E Automation: Test Framework, Test Matrix, and CI Integration
- `internals/releasing.md` — Release Process: Versioning, Release Checklist, and Publishing
- `internals/superpowers-tweakcn-theme.md` — Superpowers Spec: TweakCN Custom Theme Import Design

## `memory/` — Memory system: search, QMD engine, active memory, LanceDB, and memory wiki

7 files.

- `memory/active-memory.md` — Active Memory Plugin: Pre-Reply Memory Sub-Agent for Proactive Context Injection
- `memory/memory-config-reference.md` — Memory Configuration Reference: memorySearch, QMD, Active Memory, and Embedding Settings
- `memory/memory-honcho.md` — Honcho Memory Backend: Cloud-Based Persistent Memory for OpenClaw Agents
- `memory/memory-lancedb.md` — LanceDB Memory Plugin: Vector Database Backend for High-Scale Memory Search
- `memory/memory-qmd.md` — QMD Memory Engine: Local Search Sidecar with Reranking, Query Expansion, and Extra Index Paths
- `memory/memory-search.md` — Memory Search: Hybrid BM25 + Vector Embeddings, Embedding Providers, and memory_search Tool
- `memory/memory-wiki.md` — Memory Wiki Plugin: Structured Wiki Vault from Agent Memory with Claims and Evidence

## `nodes/` — iOS and Android nodes: camera, audio, voice wake, and media understanding

2 files.

- `nodes/media-understanding.md` — Media Understanding: Image, Video, and Audio Processing from Node Cameras and Microphones
- `nodes/nodes-overview.md` — Nodes Overview: iOS/Android Hardware Nodes for Camera, Audio, Voice, and Location

## `platforms/` — Platform-specific guides: macOS app, iOS, Android, Linux, Windows, Raspberry Pi

5 files.

- `platforms/ios-android.md` — iOS and Android Nodes: Mobile App Setup, Pairing, and Node Features
- `platforms/macos-app.md` — macOS App: Menu Bar, Permissions, Bundled Gateway, and macOS-Specific Features
- `platforms/macos-features.md` — macOS Features: Canvas, Peekaboo Screen Capture, Voice Overlay, and Voice Wake
- `platforms/macos-internals.md` — macOS Internals: Dev Setup, Code Signing, XPC, Child Process, and Logging
- `platforms/other-platforms.md` — Platform Guides: Linux, Windows, Raspberry Pi, Oracle Cloud, and DigitalOcean

## `plugins/` — Plugin SDK, architecture, building channels/providers/tools, and bundled plugins

32 files.

- `plugins/agent-tools.md` — Plugin Agent Tools: Registering and Providing Tools to the Agent
- `plugins/architecture-internals-2.md` — Plugin Internals: Message Schemas, Context Engine Plugins, and Capability Checklist
- `plugins/architecture-internals.md` — Plugin Architecture Internals: Load Pipeline, Registry, Provider Hooks, and HTTP Routes
- `plugins/building-plugins.md` — Building Plugins: Step-by-Step Guide to Creating OpenClaw Plugin Packages
- `plugins/bundles.md` — Plugin Bundles: Installing Curated Plugin Groups
- `plugins/codex-computer-use.md` — Codex Computer Use Plugin: Desktop Control via Computer Use APIs
- `plugins/codex-harness.md` — Codex Harness Plugin: OpenClaw Integration with OpenAI Codex Agent Runtime
- `plugins/community.md` — Community Plugins: Third-Party Plugin Directory and Contributing
- `plugins/compatibility.md` — Plugin Compatibility: SDK Version Support Matrix and Version Requirements
- `plugins/google-meet.md` — Google Meet Plugin: Joining Calls, Transcription, and Meeting Participation
- `plugins/hooks.md` — Plugin Hooks: Lifecycle Callbacks for Agent Events, Tool Calls, and Message Handling
- `plugins/message-presentation.md` — Message Presentation Plugin: Control UI Display and Rich Message Rendering
- `plugins/openprose.md` — OpenProse Plugin: .prose Workflow Files and Slash Commands
- `plugins/plugin-architecture.md` — Plugin Architecture: Types, Installation, and Configuration
- `plugins/plugin-manifest-2.md` — Plugin Manifest: Model Support, Model Catalog, Provider Endpoints, and Pricing
- `plugins/plugin-manifest.md` — Plugin Manifest: Declaring Channels, Providers, Tools, and All Manifest Fields
- `plugins/sdk-agent-harness.md` — SDK Agent Harness: Building Agent Runtime Adapters and Harness Plugins
- `plugins/sdk-channel-plugins.md` — SDK Channel Plugins: Building Custom Messaging Channel Integrations
- `plugins/sdk-channel-turn.md` — SDK Channel Turn: Inbound Message Handling, Turn Context, and Reply Delivery
- `plugins/sdk-entrypoints.md` — Plugin Entrypoints: Registration with the Gateway at Load Time
- ... and 12 more

## `providers/` — AI provider setup: OpenAI, Anthropic, Google, Ollama, and 50+ more

57 files.

- `providers/alibaba.md` — Alibaba Cloud Provider: Setup, Configuration, and Model Reference
- `providers/anthropic.md` — Anthropic Claude Provider: Claude Opus, Sonnet, Haiku, OAuth, and Prompt Caching
- `providers/arcee.md` — Cohere Provider: Setup, Configuration, and Model Reference
- `providers/azure-speech.md` — Azure Speech Provider: Setup, Configuration, and Model Reference
- `providers/bedrock.md` — AWS Bedrock Provider: Claude on Bedrock, Mantle Multi-Region, and IAM Setup
- `providers/cerebras.md` — Cerebras Provider: Setup, Configuration, and Model Reference
- `providers/chutes.md` — Chutes Provider: Setup, Configuration, and Model Reference
- `providers/claude-max-proxy.md` — Claude Max Proxy Provider: Setup, Configuration, and Model Reference
- `providers/cloudflare.md` — Cloudflare AI Gateway Provider: Setup, Configuration, and Model Reference
- `providers/comfy.md` — ComfyUI Provider: Setup, Configuration, and Model Reference
- `providers/deepgram.md` — Deepgram STT Provider: Setup, Configuration, and Model Reference
- `providers/deepinfra.md` — DeepInfra Provider: Setup, Configuration, and Model Reference
- `providers/deepseek.md` — DeepSeek Provider: Setup, Configuration, and Model Reference
- `providers/elevenlabs.md` — ElevenLabs TTS Provider: Setup, Configuration, and Model Reference
- `providers/fal.md` — FAL AI Provider: Setup, Configuration, and Model Reference
- `providers/fireworks.md` — Fireworks AI Provider: Setup, Configuration, and Model Reference
- `providers/github-copilot.md` — GitHub Copilot Provider: Setup, Configuration, and Model Reference
- `providers/glm.md` — GLM Z.AI Provider: Setup, Configuration, and Model Reference
- `providers/google.md` — Google Gemini Provider: Gemini Pro, Flash, Vertex AI, and Grounding Setup
- `providers/gradium.md` — Gradium Provider: Setup, Configuration, and Model Reference
- ... and 37 more

## `reference/` — API reference, templates, token usage, prompt caching, and SDK design

19 files.

- `reference/agents-default.md` — AGENTS.md Default Content: Default Agent Instructions for OpenClaw
- `reference/api-usage-costs.md` — API Usage Costs: Cost Tracking, Reporting, and Provider Pricing Reference
- `reference/auth-credential-semantics.md` — Auth Credential Semantics: Profile Resolution, Eligibility, and Routing Rules
- `reference/credits.md` — Credits, Contributors, and License
- `reference/device-models.md` — Device Models Reference: Node Hardware Model Identifiers
- `reference/docs-authoring-guide.md` — Docs Authoring Guide: Mintlify Link Rules, i18n Policy, and Documentation Structure
- `reference/memory-config-quick.md` — Memory Configuration Quick Reference: All memorySearch and QMD Fields
- `reference/modernization-plan.md` — Application Modernization Plan: Roadmap and Architectural Evolution
- `reference/prompt-caching.md` — Prompt Caching: Reducing API Costs by Caching Prompt Prefixes
- `reference/protocols.md` — Protocols Reference: Rich Output Protocol and RPC
- `reference/sdk-api-design.md` — SDK API Design: Design Principles, Conventions, and Architecture Decisions
- `reference/secretref-surfaces.md` — SecretRef Credential Surfaces: Which Configuration Fields Support Secret References
- `reference/session-management.md` — Session Management and Compaction Reference: Internals and Configuration
- `reference/testing.md` — Testing Reference: Test Suite Types, Configuration, and Running Tests
- `reference/token-use.md` — Token Usage: Counting, Tracking, and Optimizing Token Consumption
- `reference/transcript-hygiene.md` — Transcript Hygiene: Managing Session History, Sensitive Data, and File Format
- `reference/wizard-reference.md` — Wizard Configuration Reference: All Onboarding Wizard Fields and Automation Options
- `reference/workspace-templates-dev.md` — Workspace Dev Templates: AGENTS.dev.md, SOUL.dev.md, TOOLS.dev.md, and IDENTITY.dev.md
- `reference/workspace-templates.md` — Workspace Template Files: Default AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, and More

## `security/` — Threat model, security audit, network proxy, and formal verification

5 files.

- `security/contributing.md` — Contributing to the Threat Model: Format, Structure, and Submission Process
- `security/formal-verification.md` — Formal Verification: Security-Critical Code Verification in OpenClaw
- `security/network-proxy.md` — Network Proxy Security: nginx, Caddy, TLS Termination, and Reverse Proxy Config
- `security/threat-model-2.md` — Threat Model Part 2: Detailed Threat Categories and Countermeasures
- `security/threat-model.md` — OpenClaw Threat Model: Attack Vectors, Trust Boundaries, and Security Mitigations

## `tools/` — Agent tools: exec, browser, search, TTS, image/video generation, skills, and more

29 files.

- `tools/acp-agents-setup.md` — ACP Agents Setup: Configuration, Session Binding, and Delivery Model
- `tools/acp-agents.md` — ACP Agents: Running Claude Code and External Coding Agents via ACP Protocol
- `tools/agent-communication.md` — Agent Communication Tools: agent_send, Channel Messaging, and Reactions
- `tools/brave-search-setup.md` — Brave Search Setup: API Key, Plan Details, and web_search Configuration
- `tools/browser-advanced.md` — Browser Advanced: Remote CDP, Browserless, Login Persistence, WSL2, and Linux Troubleshooting
- `tools/browser.md` — Browser Tool: Chromium Control via CDP, Navigation, Forms, and Screenshots
- `tools/clawhub-lobster.md` — ClawHub and Lobster: Tool Discovery and Multi-Agent Sandbox Tool Framework
- `tools/exec-approvals.md` — Exec Approvals: Human Confirmation Gates for Shell Command Execution
- `tools/exec.md` — Exec Tool: Shell Command Execution, Elevated Mode, and Code Execution
- `tools/file-patching.md` — File Patching: apply_patch Tool and Diffs for Code Modification
- `tools/image-generation.md` — Image Generation Tool: DALL-E, fal.ai, ComfyUI Providers and Configuration
- `tools/llm-task.md` — LLM Task Tool: Discrete LLM Calls for Classification, Extraction, and Summarization
- `tools/media-overview.md` — Media Tools Overview: Image, Video, Audio Generation and Understanding
- `tools/misc-tools.md` — Miscellaneous Tools: btw, Capability Cookbook, Loop Detection, Trajectory, TokenJuice
- `tools/music-generation.md` — Music Generation Tool: Audio Generation and Provider Configuration
- `tools/pdf.md` — PDF Tool: Reading and Extracting Text from PDF Files
- `tools/perplexity-setup.md` — Perplexity Search Setup: API Key, Sonar Model, and OpenRouter Compatibility
- `tools/plugin-tool.md` — Plugin Tool: Agent Interaction with Plugin-Provided Tools and Capabilities
- `tools/skills.md` — Skills: Contextual Instruction Files, Loading Skills, and Creating Custom Skills
- `tools/slash-commands-advanced.md` — Slash Commands Advanced: Plugin Commands, Custom Commands, and Channel Behavior
- ... and 9 more

## `troubleshooting/` — Troubleshooting: FAQ, first-run issues, model errors, debugging, and testing

13 files.

- `troubleshooting/debug-and-flags.md` — Node Debug and Diagnostic Flags: Debugging Node Issues and Gateway Diagnostics
- `troubleshooting/debugging.md` — Debugging Guide: Log Levels, Trace Mode, Diagnostic Commands, and Issue Reporting
- `troubleshooting/environment-variables.md` — Environment Variables Reference: All OpenClaw Env Vars and Their Effects
- `troubleshooting/faq-2.md` — FAQ Part 2: Sessions, Remote Gateways, Security, and Media Questions
- `troubleshooting/faq-first-run.md` — First Run FAQ: Common Setup Issues, API Keys, and Initial Configuration Problems
- `troubleshooting/faq-models.md` — Models FAQ: Choosing Models, API Keys, Rate Limits, and Model-Specific Issues
- `troubleshooting/faq.md` — FAQ: Common Questions About OpenClaw Setup, Config, Models, and Channels
- `troubleshooting/general-troubleshooting.md` — General Troubleshooting: Entry Points for Gateway, Channel, Model, and Tool Issues
- `troubleshooting/gpt55-codex-parity.md` — GPT-5 and Codex Agentic Parity: Compatibility Notes and Maintainer Guidance
- `troubleshooting/logging-guide.md` — Logging Guide: File Logs, Console Output, Log Levels, and CLI Tailing
- `troubleshooting/scripts-and-help.md` — Help Index and Scripts: Helper Scripts and Utility Reference
- `troubleshooting/testing-live.md` — Live Testing: Testing with Real Providers, Models, and Channels
- `troubleshooting/testing.md` — Testing Guide: Running Unit, Integration, E2E, and Live Tests

## `web-ui/` — Web interfaces: Control UI dashboard, WebChat, and terminal TUI

2 files.

- `web-ui/control-ui.md` — Control UI: Browser Dashboard for Chat, Config, Sessions, and Logs
- `web-ui/webchat-dashboard-tui.md` — WebChat, Dashboard, and TUI: Web and Terminal Interfaces for OpenClaw


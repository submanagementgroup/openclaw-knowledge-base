---
domain: troubleshooting
topic: "FAQ: Common Questions About OpenClaw Setup, Config, Models, and Channels"
type: reference
keywords:
  - FAQ
  - frequently asked questions
  - common questions
  - troubleshooting FAQ
related:
  - troubleshooting/faq-first-run
  - troubleshooting/faq-models
source: help/faq.md
---

Frequently asked questions about OpenClaw: setup, configuration, models, channels, and common errors.

Quick answers plus deeper troubleshooting for real-world setups (local dev, VPS, multi-agent, OAuth/API keys, model failover). For runtime diagnostics, see [Troubleshooting](/gateway/troubleshooting). For the full config reference, see [Configuration](/gateway/configuration).

## First 60 seconds if something is broken

1. **Quick status (first check)**

   ```bash
   openclaw status
   ```

   Fast local summary: OS + update, gateway/service reachability, agents/sessions, provider config + runtime issues (when gateway is reachable).

2. **Pasteable report (safe to share)**

   ```bash
   openclaw status --all
   ```

   Read-only diagnosis with log tail (tokens redacted).

3. **Daemon + port state**

   ```bash
   openclaw gateway status
   ```

   Shows supervisor runtime vs RPC reachability, the probe target URL, and which config the service likely used.

4. **Deep probes**

   ```bash
   openclaw status --deep
   ```

   Runs a live gateway health probe, including channel probes when supported
   (requires a reachable gateway). See [Health](/gateway/health).

5. **Tail the latest log**

   ```bash
   openclaw logs --follow
   ```

   If RPC is down, fall back to:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   File logs are separate from service logs; see [Logging](/logging) and [Troubleshooting](/gateway/troubleshooting).

6. **Run the doctor (repairs)**

   ```bash
   openclaw doctor
   ```

   Repairs/migrates config/state + runs health checks. See [Doctor](/gateway/doctor).

7. **Gateway snapshot**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   Asks the running gateway for a full snapshot (WS-only). See [Health](/gateway/health).

## Quick start and first-run setup

First-run Q&A — install, onboard, auth routes, subscriptions, initial failures —
lives on the [First-run FAQ](/help/faq-first-run).

## What is OpenClaw?

    OpenClaw is a personal AI assistant you run on your own devices. It replies on the messaging surfaces you already use (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat, and bundled channel plugins such as QQ Bot) and can also do voice + a live Canvas on supported platforms. The **Gateway** is the always-on control plane; the assistant is the product.

    OpenClaw is not "just a Claude wrapper." It's a **local-first control plane** that lets you run a
    capable assistant on **your own hardware**, reachable from the chat apps you already use, with
    stateful sessions, memory, and tools - without handing control of your workflows to a hosted
    SaaS.

    Highlights:

    - **Your devices, your data:** run the Gateway wherever you want (Mac, Linux, VPS) and keep the
      workspace + session history local.
    - **Real channels, not a web sandbox:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc,
      plus mobile voice and Canvas on supported platforms.
    - **Model-agnostic:** use Anthropic, OpenAI, MiniMax, OpenRouter, etc., with per-agent routing
      and failover.
    - **Local-only option:** run local models so **all data can stay on your device** if you want.
    - **Multi-agent routing:** separate agents per channel, account, or task, each with its own
      workspace and defaults.
    - **Open source and hackable:** inspect, extend, and self-host without vendor lock-in.

    Docs: [Gateway](/gateway), [Channels](/channels), [Multi-agent](/concepts/multi-agent),
    [Memory](/concepts/memory).

    Good first projects:

    - Build a website (WordPress, Shopify, or a simple static site).
    - Prototype a mobile app (outline, screens, API plan).
    - Organize files and folders (cleanup, naming, tagging).
    - Connect Gmail and automate summaries or follow ups.

    It can handle large tasks, but it works best when you split them into phases and
    use sub agents for parallel work.

    Everyday wins usually look like:

    - **Personal briefings:** summaries of inbox, calendar, and news you care about.
    - **Research and drafting:** quick research, summaries, and first drafts for emails or docs.
    - **Reminders and follow ups:** cron or heartbeat driven nudges and checklists.
    - **Browser automation:** filling forms, collecting data, and repeating web tasks.
    - **Cross device coordination:** send a task from your phone, let the Gateway run it on a server, and get the result back in chat.

    Yes for **research, qualification, and drafting**. It can scan sites, build shortlists,
    summarize prospects, and write outreach or ad copy drafts.

    For **outreach or ad runs**, keep a human in the loop. Avoid spam, follow local laws and
    platform policies, and review anything before it is sent. The safest pattern is to let
    OpenClaw draft and you approve.

    Docs: [Security](/gateway/security).

    OpenClaw is a **personal assistant** and coordination layer, not an IDE replacement. Use
    Claude Code or Codex for the fastest direct coding loop inside a repo. Use OpenClaw when you
    want durable memory, cross-device access, and tool orchestration.

    Advantages:

    - **Persistent memory + workspace** across sessions
    - **Multi-platform access** (WhatsApp, Telegram, TUI, WebChat)
    - **Tool orchestration** (browser, files, scheduling, hooks)
    - **Always-on Gateway** (run on a VPS, interact from anywhere)
    - **Nodes** for local browser/screen/camera/exec

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

## Skills and automation

    Use managed overrides instead of editing the repo copy. Put your changes in `~/.openclaw/skills/<name>/SKILL.md` (or add a folder via `skills.load.extraDirs` in `~/.openclaw/openclaw.json`). Precedence is `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bundled → `skills.load.extraDirs`, so managed overrides still win over bundled skills without touching git. If you need the skill installed globally but only visible to some agents, keep the shared copy in `~/.openclaw/skills` and control visibility with `agents.defaults.skills` and `agents.list[].skills`. Only upstream-worthy edits should live in the repo and go out as PRs.

    Yes. Add extra directories via `skills.load.extraDirs` in `~/.openclaw/openclaw.json` (lowest precedence). Default precedence is `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bundled → `skills.load.extraDirs`. `clawhub` installs into `./skills` by default, which OpenClaw treats as `<workspace>/skills` on the next session. If the skill should only be visible to certain agents, pair that with `agents.defaults.skills` or `agents.list[].skills`.

    Today the supported patterns are:

    - **Cron jobs**: isolated jobs can set a `model` override per job.
    - **Sub-agents**: route tasks to separate agents with different default models.
    - **On-demand switch**: use `/model` to switch the current session model at any time.

    See [Cron jobs](/automation/cron-jobs), [Multi-Agent Routing](/concepts/multi-agent), and [Slash commands](/tools/slash-commands).

    Use **sub-agents** for long or parallel tasks. Sub-agents run in their own session,
    return a summary, and keep your main chat responsive.

    Ask your bot to "spawn a sub-agent for this task" or use `/subagents`.
    Use `/status` in chat to see what the Gateway is doing right now (and whether it is busy).

    Token tip: long tasks and sub-agents both consume tokens. If cost is a concern, set a
    cheaper model for sub-agents via `agents.defaults.subagents.model`.

    Docs: [Sub-agents](/tools/subagents), [Background Tasks](/automation/tasks).

    Use thread bindings. You can bind a Discord thread to a subagent or session target so follow-up messages in that thread stay on that bound session.

    Basic flow:

    - Spawn with `sessions_spawn` using `thread: true` (and optionally `mode: "session"` for persistent follow-up).
    - Or manually bind with `/focus <target>`.
    - Use `/agents` to inspect binding state.
    - Use `/session idle <duration|off>` and `/session max-age <duration|off>` to control auto-unfocus.
    - Use `/unfocus` to detach the thread.

    Required config:

    - Global defaults: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Discord overrides: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Auto-bind on spawn: set `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Docs: [Sub-agents](/tools/subagents), [Discord](/channels/discord), [Configuration Reference](/gateway/configuration-reference), [Slash commands](/tools/slash-commands).

    Check the resolved requester route first:

    - Completion-mode subagent delivery prefers any bound thread or conversation rou

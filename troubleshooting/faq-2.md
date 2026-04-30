---
domain: troubleshooting
topic: "FAQ Part 2: Sessions, Remote Gateways, Security, and Media Questions"
type: reference
keywords:
  - FAQ sessions
  - FAQ remote gateway
  - FAQ security
  - FAQ media
  - FAQ miscellaneous
related:
  - troubleshooting/faq
source: help/faq.md
---

FAQ continued: sessions, remote gateways, security, media, and miscellaneous questions.

te when one exists.
    - If the completion origin only carries a channel, OpenClaw falls back to the requester session's stored route (`lastChannel` / `lastTo` / `lastAccountId`) so direct delivery can still succeed.
    - If neither a bound route nor a usable stored route exists, direct delivery can fail and the result falls back to queued session delivery instead of posting immediately to chat.
    - Invalid or stale targets can still force queue fallback or final delivery failure.
    - If the child's last visible assistant reply is the exact silent token `NO_REPLY` / `no_reply`, or exactly `ANNOUNCE_SKIP`, OpenClaw intentionally suppresses the announce instead of posting stale earlier progress.
    - If the child timed out after only tool calls, the announce can collapse that into a short partial-progress summary instead of replaying raw tool output.

    Debug:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Docs: [Sub-agents](/tools/subagents), [Background Tasks](/automation/tasks), [Session Tools](/concepts/session-tool).

    Cron runs inside the Gateway process. If the Gateway is not running continuously,
    scheduled jobs will not run.

    Checklist:

    - Confirm cron is enabled (`cron.enabled`) and `OPENCLAW_SKIP_CRON` is not set.
    - Check the Gateway is running 24/7 (no sleep/restarts).
    - Verify timezone settings for the job (`--tz` vs host timezone).

    Debug:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Docs: [Cron jobs](/automation/cron-jobs), [Automation & Tasks](/automation).

    Check the delivery mode first:

    - `--no-deliver` / `delivery.mode: "none"` means no runner fallback send is expected.
    - Missing or invalid announce target (`channel` / `to`) means the runner skipped outbound delivery.
    - Channel auth failures (`unauthorized`, `Forbidden`) mean the runner tried to deliver but credentials blocked it.
    - A silent isolated result (`NO_REPLY` / `no_reply` only) is treated as intentionally non-deliverable, so the runner also suppresses queued fallback delivery.

    For isolated cron jobs, the agent can still send directly with the `message`
    tool when a chat route is available. `--announce` only controls the runner
    fallback path for final text that the agent did not already send.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Docs: [Cron jobs](/automation/cron-jobs), [Background Tasks](/automation/tasks).

    That is usually the live model-switch path, not duplicate scheduling.

    Isolated cron can persist a runtime model handoff and retry when the active
    run throws `LiveSessionModelSwitchError`. The retry keeps the switched
    provider/model, and if the switch carried a new auth profile override, cron
    persists that too before retrying.

    Related selection rules:

    - Gmail hook model override wins first when applicable.
    - Then per-job `model`.
    - Then any stored cron-session model override.
    - Then the normal agent/default model selection.

    The retry loop is bounded. After the initial attempt plus 2 switch retries,
    cron aborts instead of looping forever.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Docs: [Cron jobs](/automation/cron-jobs), [cron CLI](/cli/cron).

    Use native `openclaw skills` commands or drop skills into your workspace. The macOS Skills UI isn't available on Linux.
    Browse skills at [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    Native `openclaw skills install` writes into the active workspace `skills/`
    directory. Install the separate `clawhub` CLI only if you want to publish or
    sync your own skills. For shared installs across agents, put the skill under
    `~/.openclaw/skills` and use `agents.defaults.skills` or
    `agents.list[].skills` if you want to narrow which agents can see it.

    Yes. Use the Gateway scheduler:

    - **Cron jobs** for scheduled or recurring tasks (persist across restarts).
    - **Heartbeat** for "main session" periodic checks.
    - **Isolated jobs** for autonomous agents that post summaries or deliver to chats.

    Docs: [Cron jobs](/automation/cron-jobs), [Automation & Tasks](/automation),
    [Heartbeat](/gateway/heartbeat).

    Not directly. macOS skills are gated by `metadata.openclaw.os` plus required binaries, and skills only appear in the system prompt when they are eligible on the **Gateway host**. On Linux, `darwin`-only skills (like `apple-notes`, `apple-reminders`, `things-mac`) will not load unless you override the gating.

    You have three supported patterns:

    **Option A - run the Gateway on a Mac (simplest).**
    Run the Gateway where the macOS binaries exist, then connect from Linux in [remote mode](#gateway-ports-already-running-and-remote-mode) or over Tailscale. The skills load normally because the Gateway host is macOS.

    **Option B - use a macOS node (no SSH).**
    Run the Gateway on Linux, pair a macOS node (menubar app), and set **Node Run Commands** to "Always Ask" or "Always Allow" on the Mac. OpenClaw can treat macOS-only skills as eligible when the required binaries exist on the node. The agent runs those skills via the `nodes` tool. If you choose "Always Ask", approving "Always Allow" in the prompt adds that command to the allowlist.

    **Option C - proxy macOS binaries over SSH (advanced).**
    Keep the Gateway on Linux, but make the required CLI binaries resolve to SSH wrappers that run on a Mac. Then override the skill to allow Linux so it stays eligible.

    1. Create an SSH wrapper for the binary (example: `memo` for Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Put the wrapper on `PATH` on the Linux host (for example `~/bin/memo`).
    3. Override the skill metadata (workspace or `~/.openclaw/skills`) to allow Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Start a new session so the skills snapshot refreshes.

    Not built-in today.

    Options:

    - **Custom skill / plugin:** best for reliable API access (Notion/HeyGen both have APIs).
    - **Browser automation:** works without code but is slower and more fragile.

    If you want to keep context per client (agency workflows), a simple pattern is:

    - One Notion page per client (context + preferences + active work).
    - Ask the agent to fetch that page at the start of a session.

    If you want a native integration, open a feature request or build a skill
    targeting those APIs.

    Install skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Native installs land in the active workspace `skills/` directory. For shared skills across agents, place them in `~/.openclaw/skills/<name>/SKILL.md`. If only some agents should see a shared install, configure `agents.defaults.skills` or `agents.list[].skills`. Some skills expect binaries installed via Homebrew; on Linux that means Linuxbrew (see the Homebrew Linux FAQ entry above). See [Skills](/tools/skills), [Skills config](/tools/skills-config), and [ClawHub](/tools/clawhub).

    Use the built-in `user` browser profile, which attaches through Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    If you want a custom name, create an explicit MCP profile:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    This path can use the local host browser or a connected browser node. If the Gateway runs elsewhere, either run a node host on the browser machine or use remote CDP instead.

    Current limits on `existing-session` / `user`:

    - actions are ref-driven, not CSS-selector driven
    - uploads require `ref` / `inputRef` and currently support one file at a time
    - `responsebody`, PDF export, download interception, and batch actions still need a managed browser or raw CDP profile

## Sandboxing and memory

    Yes. See [Sandboxing](/gateway/sandboxing). For Docker-specific setup (full gateway in Docker or sandbox images), see [Docker](/install/docker).

    Th

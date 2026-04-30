---
domain: troubleshooting
topic: "First Run FAQ: Common Setup Issues, API Keys, and Initial Configuration Problems"
type: troubleshooting
keywords:
  - first run
  - setup issues
  - first-run FAQ
  - initial setup
  - API key errors
  - getting started errors
related:
  - getting-started/quickstart
  - troubleshooting/faq
source: help/faq-first-run.md
---

First-run troubleshooting: common issues encountered during initial setup and first use of OpenClaw.

Quick-start and first-run Q&A. For everyday operations, models, auth, sessions,
and troubleshooting see the main [FAQ](/help/faq).

## Quick start and first-run setup

    Use a local AI agent that can **see your machine**. That is far more effective than asking
    in Discord, because most "I'm stuck" cases are **local config or environment issues** that
    remote helpers cannot inspect.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    These tools can read the repo, run commands, inspect logs, and help fix your machine-level
    setup (PATH, services, permissions, auth files). Give them the **full source checkout** via
    the hackable (git) install:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    This installs OpenClaw **from a git checkout**, so the agent can read the code + docs and
    reason about the exact version you are running. You can always switch back to stable later
    by re-running the installer without `--install-method git`.

    Tip: ask the agent to **plan and supervise** the fix (step-by-step), then execute only the
    necessary commands. That keeps changes small and easier to audit.

    If you discover a real bug or fix, please file a GitHub issue or send a PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Start with these commands (share outputs when asking for help):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    What they do:

    - `openclaw status`: quick snapshot of gateway/agent health + basic config.
    - `openclaw models status`: checks provider auth + model availability.
    - `openclaw doctor`: validates and repairs common config/state issues.

    Other useful CLI checks: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Quick debug loop: [First 60 seconds if something is broken](#first-60-seconds-if-something-is-broken).
    Install docs: [Install](/install), [Installer flags](/install/installer), [Updating](/install/updating).

    Common heartbeat skip reasons:

    - `quiet-hours`: outside the configured active-hours window
    - `empty-heartbeat-file`: `HEARTBEAT.md` exists but only contains blank/header-only scaffolding
    - `no-tasks-due`: `HEARTBEAT.md` task mode is active but none of the task intervals are due yet
    - `alerts-disabled`: all heartbeat visibility is disabled (`showOk`, `showAlerts`, and `useIndicator` are all off)

    In task mode, due timestamps are only advanced after a real heartbeat run
    completes. Skipped runs do not mark tasks as completed.

    Docs: [Heartbeat](/gateway/heartbeat), [Automation & Tasks](/automation).

    The repo recommends running from source and using onboarding:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    The wizard can also build UI assets automatically. After onboarding, you typically run the Gateway on port **18789**.

    From source (contributors/dev):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    If you don't have a global install yet, run it via `pnpm openclaw onboard`.

    The wizard opens your browser with a clean (non-tokenized) dashboard URL right after onboarding and also prints the link in the summary. Keep that tab open; if it didn't launch, copy/paste the printed URL on the same machine.

    **Localhost (same machine):**

    - Open `http://127.0.0.1:18789/`.
    - If it asks for shared-secret auth, paste the configured token or password into Control UI settings.
    - Token source: `gateway.auth.token` (or `OPENCLAW_GATEWAY_TOKEN`).
    - Password source: `gateway.auth.password` (or `OPENCLAW_GATEWAY_PASSWORD`).
    - If no shared secret is configured yet, generate a token with `openclaw doctor --generate-gateway-token`.

    **Not on localhost:**

    - **Tailscale Serve** (recommended): keep bind loopback, run `openclaw gateway --tailscale serve`, open `https://<magicdns>/`. If `gateway.auth.allowTailscale` is `true`, identity headers satisfy Control UI/WebSocket auth (no pasted shared secret, assumes trusted gateway host); HTTP APIs still require shared-secret auth unless you deliberately use private-ingress `none` or trusted-proxy HTTP auth.
      Bad concurrent Serve auth attempts from the same client are serialized before the failed-auth limiter records them, so the second bad retry can already show `retry later`.
    - **Tailnet bind**: run `openclaw gateway --bind tailnet --token "<token>"` (or configure password auth), open `http://<tailscale-ip>:18789/`, then paste the matching shared secret in dashboard settings.
    - **Identity-aware reverse proxy**: keep the Gateway behind a trusted proxy, configure `gateway.auth.mode: "trusted-proxy"`, then open the proxy URL. Same-host loopback proxies require explicit `gateway.auth.trustedProxy.allowLoopback = true`.
    - **SSH tunnel**: `ssh -N -L 18789:127.0.0.1:18789 user@host` then open `http://127.0.0.1:18789/`. Shared-secret auth still applies over the tunnel; paste the configured token or password if prompted.

    See [Dashboard](/web/dashboard) and [Web surfaces](/web) for bind modes and auth details.

    They control different layers:

    - `approvals.exec`: forwards approval prompts to chat destinations
    - `channels.<channel>.execApprovals`: makes that channel act as a native approval client for exec approvals

    The host exec policy is still the real approval gate. Chat config only controls where approval
    prompts appear and how people can answer them.

    In most setups you do **not** need both:

    - If the chat already supports commands and replies, same-chat `/approve` works through the shared path.
    - If a supported native channel can infer approvers safely, OpenClaw now auto-enables DM-first native approvals when `channels.<channel>.execApprovals.enabled` is unset or `"auto"`.
    - When native approval cards/buttons are available, that native UI is the primary path; the agent should only include a manual `/approve` command if the tool result says chat approvals are unavailable or manual approval is the only path.
    - Use `approvals.exec` only when prompts must also be forwarded to other chats or explicit ops rooms.
    - Use `channels.<channel>.execApprovals.target: "channel"` or `"both"` only when you explicitly want approval prompts posted back into the originating room/topic.
    - Plugin approvals are separate again: they use same-chat `/approve` by default, optional `approvals.plugin` forwarding, and only some native channels keep plugin-approval-native handling on top.

    Short version: forwarding is for routing, native client config is for richer channel-specific UX.
    See [Exec Approvals](/tools/exec-approvals).

    Node **>= 22** is required. `pnpm` is recommended. Bun is **not recommended** for the Gateway.

    Yes. The Gateway is lightweight - docs list **512MB-1GB RAM**, **1 core**, and about **500MB**
    disk as enough for personal use, and note that a **Raspberry Pi 4 can run it**.

    If you want extra headroom (logs, media, other services), **2GB is recommended**, but it's
    not a hard minimum.

    Tip: a small Pi/VPS can host the Gateway, and you can pair **nodes** on your laptop/phone for
    local screen/camera/canvas or command execution. See [Nodes](/nodes).

    Short version: it works, but expect rough edges.

    - Use a **64-bit** OS and keep Node >= 22.
    - Prefer the **hackable (git) install** so you can see logs and update fast.
    - Start without channels/skills, then add them one by one.
    - If you hit weird binary issues, it is usually an **ARM compatibility** problem.

    Docs: [Linux](/platforms/linux), [Install](/install).

    That screen depends on the Gateway being reachable and authenticated. The TUI also sends
    "Wake up, my friend!" automatically on first hatch. If you see that line with **no reply**
    and tokens stay at 0, the agent never ran.

    1. Restart the Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. Check status + auth:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. If it still hangs, run:

    ```bash
    openclaw doctor
    ```

    If the Gateway is remote, ensure the tunnel/Tailscale connection is up and that the UI
    is pointed at the right Gateway. See [Remote access](/gateway/remote).

    Yes. Copy the **state directory** and **workspace**, then run Doctor once. This
    keeps your bot "exactly the same" (memory, session history, auth,

---
domain: tools
topic: "Browser Tool: Chromium Control via CDP, Navigation, Forms, and Screenshots"
type: procedure
keywords:
  - browser tool
  - CDP
  - Chromium
  - web automation
  - browser control
  - browser CDP
  - web scraping
related:
  - tools/browser-advanced
  - tools/web-fetch
  - gateway/sandboxing
source: tools/browser.md
---

The browser tool gives the agent control of a Chromium browser via Chrome DevTools Protocol (CDP). The agent can navigate, click, fill forms, and capture screenshots.

## Quick Start

```json5
{
  tools: {
    enabled: ["browser"],
  }
}
```

The browser auto-starts when the agent needs it (requires Chrome/Chromium installed).

## Browser Basics

OpenClaw can run a **dedicated Chrome/Brave/Edge/Chromium profile** that the agent controls.
It is isolated from your personal browser and is managed through a small local
control service inside the Gateway (loopback only).

Beginner view:

- Think of it as a **separate, agent-only browser**.
- The `openclaw` profile does **not** touch your personal browser profile.
- The agent can **open tabs, read pages, click, and type** in a safe lane.
- The built-in `user` profile attaches to your real signed-in Chrome session via Chrome MCP.

## What you get

- A separate browser profile named **openclaw** (orange accent by default).
- Deterministic tab control (list/open/focus/close).
- Agent actions (click/type/drag/select), snapshots, screenshots, PDFs.
- A bundled `browser-automation` skill that teaches agents the snapshot,
  stable-tab, stale-ref, and manual-blocker recovery loop when the browser
  plugin is enabled.
- Optional multi-profile support (`openclaw`, `work`, `remote`, ...).

This browser is **not** your daily driver. It is a safe, isolated surface for
agent automation and verification.

## Quick start

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw doctor --deep
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

If you get “Browser disabled”, enable it in config (see below) and restart the
Gateway.

If `openclaw browser` is missing entirely, or the agent says the browser tool
is unavailable, jump to [Missing browser command or tool](/tools/browser#missing-browser-command-or-tool).

## Plugin control

The default `browser` tool is a bundled plugin. Disable it to replace it with another plugin that registers the same `browser` tool name:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

Defaults need both `plugins.entries.browser.enabled` **and** `browser.enabled=true`. Disabling only the plugin removes the `openclaw browser` CLI, `browser.request` gateway method, agent tool, and control service as one unit; your `browser.*` config stays intact for a replacement.

Browser config changes require a Gateway restart so the plugin can re-register its service.

## Agent guidance

Tool-profile note: `tools.profile: "coding"` includes `web_search` and
`web_fetch`, but it does not include the full `browser` tool. If the agent or a
spawned sub-agent should use browser automation, add browser at the profile
stage:

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

For a single agent, use `agents.list[].tools.alsoAllow: ["browser"]`.
`tools.subagents.tools.allow: ["browser"]` alone is not enough because sub-agent
policy is applied after profile filtering.

The browser plugin ships two levels of agent guidance:

- The `browser` tool description carries the compact always-on contract: pick
  the right profile, keep refs on the same tab, use `tabId`/labels for tab
  targeting, and load the browser skill for multi-step work.
- The bundled `browser-automation` skill carries the longer operating loop:
  check status/tabs first, label task tabs, snapshot before acting, resnapshot
  after UI changes, recover stale refs once, and report login/2FA/captcha or
  camera/microphone blockers as manual action instead of guessing.

Plugin-bundled skills are listed in the agent's available skills when the
plugin is enabled. The full skill instructions are loaded on demand, so routine
turns do not pay the full token cost.

## Missing browser command or tool

If `openclaw browser` is unknown after an upgrade, `browser.request` is missing, or the agent reports the browser tool as unavailable, the usual cause is a `plugins.allow` list that omits `browser` and no root `browser` config block exists. Add it:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

An explicit root `browser` block, for example `browser.enabled=true` or `browser.profiles.<name>`, activates the bundled browser plugin even under a restrictive `plugins.allow`, matching channel config behavior. `plugins.entries.browser.enabled=true` and `tools.alsoAllow: ["browser"]` do not substitute for allowlist membership by themselves. Removing `plugins.allow` entirely also restores the default.

## Profiles: `openclaw` vs `user`

- `openclaw`: managed, isolated browser (no extension required).
- `user`: built-in Chrome MCP attach profile for your **real signed-in Chrome**
  session.

For agent browser tool calls:

- Default: use the isolated `openclaw` browser.
- Prefer `profile="user"` when existing logged-in sessions matter and the user
  is at the computer to click/approve any attach prompt.
- `profile` is the explicit override when you want a specific browser mode.

Set `browser.defaultProfile: "openclaw"` if you want managed mode b

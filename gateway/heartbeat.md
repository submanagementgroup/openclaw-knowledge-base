---
domain: gateway
topic: "Heartbeat: Periodic Agent Turns, HEARTBEAT.md, Active Hours, and heartbeat vs cron"
type: concept
keywords:
  - heartbeat
  - periodic agent runs
  - HEARTBEAT.md
  - heartbeat.every
  - heartbeat target
  - active hours
related:
  - automation/cron-jobs
  - automation/automation-overview
  - gateway/config-agents-reference
source: gateway/heartbeat.md
---

Heartbeat runs periodic agent turns in the main session so the model can surface pending items without user prompting. It is scheduled (default 30m) and runs independently of cron jobs.

## Quick Setup

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",           // cadence (or "1h", "2h", etc.)
        target: "last",         // route replies to last active channel
        // target: "none"       // (default) silent heartbeat, no outbound reply
      }
    }
  }
}
```

## Heartbeat vs Cron

| | Heartbeat | Cron |
|--|-----------|------|
| Runs in | Main session | Isolated session or main |
| Creates task record | No | Yes |
| Suitable for | Ambient check-ins | Scheduled discrete jobs |
| State access | Full main session history | Configurable context |

## HEARTBEAT.md

Create `~/.openclaw/workspace/HEARTBEAT.md` with a checklist or tasks block. The agent reads this file on every heartbeat turn:

```markdown
## Heartbeat tasks
- [ ] Check email for urgent items
- [ ] Review open commitments
- [ ] Post daily standup if 9am
```

## Active Hours Restriction

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        activeHours: { start: "08:00", end: "22:00", tz: "America/Los_Angeles" }
      }
    }
  }
}
```

**Heartbeat vs cron?** See [Automation & Tasks](/automation) for guidance on when to use each.

Heartbeat runs **periodic agent turns** in the main session so the model can surface anything that needs attention without spamming you.

Heartbeat is a scheduled main-session turn — it does **not** create [background task](/automation/tasks) records. Task records are for detached work (ACP runs, subagents, isolated cron jobs).

Troubleshooting: [Scheduled Tasks](/automation/cron-jobs#troubleshooting)

## Quick start (beginner)

    Leave heartbeats enabled (default is `30m`, or `1h` for Anthropic OAuth/token auth, including Claude CLI reuse) or set your own cadence.

    Create a tiny `HEARTBEAT.md` checklist or `tasks:` block in the agent workspace.

    `target: "none"` is the default; set `target: "last"` to route to the last contact.

    - Enable heartbeat reasoning delivery for transparency.
    - Use lightweight bootstrap context if heartbeat runs only need `HEARTBEAT.md`.
    - Enable isolated sessions to avoid sending full conversation history each heartbeat.
    - Restrict heartbeats to active hours (local time).

Example config:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explicit delivery to last contact (default is "none")
        directPolicy: "allow", // default: allow direct/DM targets; set "block" to suppress
        lightContext: true, // optional: only inject HEARTBEAT.md from bootstrap files
        isolatedSession: true, // optional: fresh session each run (no conversation history)
        skipWhenBusy: true, // optional: also defer when subagent or nested lanes are busy
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // optional: send separate `Reasoning:` message too
      },
    },
  },
}
```

## Defaults

- Interval: `30m` (or `1h` when Anthropic OAuth/token auth is the detected auth mode, including Claude CLI reuse). Set `agents.defaults.heartbeat.every` or per-agent `agents.list[].heartbeat.every`; use `0m` to disable.
- Prompt body (configurable via `agents.defaults.heartbeat.prompt`): `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- The heartbeat prompt is sent **verbatim** as the user message. The system prompt includes a "Heartbeat" section only when heartbeats are enabled for the default agent, and the run is flagged internally.
- When heartbeats are disabled with `0m`, normal runs also omit `HEARTBEAT.md` from bootstrap context so the model does not see heartbeat-only instructions.
- Active hours (`heartbeat.activeHours`) are checked in the configured timezone. Outside the window, heartbeats are skipped until the next tick inside the window.
- Heartbeats automatically defer while cron work is active or queued. Set `heartbeat.skipWhenBusy: true` to defer on extra busy lanes (subagent or nested command work) as well; this is useful for local Ollama and other constrained single-runtime hosts.

## What the heartbeat prompt is for

The default prompt is intentionally broad:

- **Background tasks**: "Consider outstanding tasks" nudges the agent to review follow-ups (inbox, calendar, reminders, queued work) and surface anything urgent.
- **Human check-in**: "Checkup sometimes on your human during day time" nudges an occasional lightweight "anything you need?" message, but avoids night-time spam by using your configured local timezone (see [Timezone](/concepts/timezone)).

Heartbeat can react to completed [background tasks](/automation/tasks), but a heartbeat run itself does not create a task record.

If you want a heartbeat to do something very specific (e.g. "check Gmail PubSub stats" or "verify gateway health"), set `agents.defaults.heartbeat.prompt` (or `agents.list[].heartbeat.prompt`) to a custom body (sent verbatim).

## Response contract

- If nothing needs attention,

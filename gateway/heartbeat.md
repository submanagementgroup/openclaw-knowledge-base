---
domain: gateway
topic: "Heartbeat: Scheduled Agent Runs"
type: procedure
keywords:
  - heartbeat
  - scheduled runs
  - cron heartbeat
  - heartbeat.every
  - autonomous agent
  - background tasks
source: gateway/heartbeat.md
related:
  - automation/cron
---

> **Note:** **Heartbeat vs cron?** See [Automation](/automation) for guidance on when to use each.


Heartbeat runs **periodic agent turns** in the main session so the model can surface anything that needs attention without spamming you.

Heartbeat is a scheduled main-session turn — it does **not** create [background task](/automation/tasks) records. Task records are for detached work (ACP runs, subagents, isolated cron jobs).

Troubleshooting: [Scheduled Tasks](/automation/cron-jobs#troubleshooting)

## Quick start (beginner)

**Pick a cadence**

Leave heartbeats enabled (default is `30m`, or `1h` for Anthropic OAuth/token auth, including Claude CLI reuse) or set your own cadence.
  
**Add HEARTBEAT.md (optional)**

Create a tiny `HEARTBEAT.md` checklist or `tasks:` block in the agent workspace.
  
**Decide where heartbeat messages should go**

`target: "none"` is the default; set `target: "last"` to route to the last contact.
  
**Optional tuning**

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

- If nothing needs attention, reply with **`HEARTBEAT_OK`**.
- Tool-capable heartbeat runs may instead call `heartbeat_respond` with `notify: false` for no visible update, or `notify: true` plus `notificationText` for an alert. When present, the structured tool response takes precedence over the text fallback.
- During heartbeat runs, OpenClaw treats `HEARTBEAT_OK` as an ack when it appears at the **start or end** of the reply. The token is stripped and the reply is dropped if the remaining content is **≤ `ackMaxChars`** (default: 300).
- If `HEARTBEAT_OK` appears in the **middle** of a reply, it is not treated specially.
- For alerts, **do not** include `HEARTBEAT_OK`; return only the alert text.

Outside heartbeats, stray `HEARTBEAT_OK` at the start/end of a message is stripped and logged; a message that is only `HEARTBEAT_OK` is dropped.

## Config

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // default: 30m (0m disables)
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // default: false (deliver separate Reasoning: message when available)
        lightContext: false, // default: false; true keeps only HEARTBEAT.md from workspace bootstrap files
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        skipWhenBusy: false, // default: false; true also waits for subagent/nested lanes
        target: "last", // default: none | options: last | none | <channel id> (core or plugin, e.g. "imessage")
        to: "+15551234567", // optional channel-specific override
        accountId: "ops-bot", // optional multi-account channel id
        prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        ackMaxChars: 300, // max chars allowed after HEARTBEAT_OK
      },
    },
  },
}
```

### Scope and precedence

- `agents.defaults.heartbeat` sets global heartbeat behavior.
- `agents.list[].heartbeat` merges on top; if any agent has a `heartbeat` block, **only those agents** run heartbeats.
- `channels.defaults.heartbeat` sets visibility defaults for all channels.
- `channels.<channel>.heartbeat` overrides channel defaults.
- `channels.<channel>.accounts.<id>.heartbeat` (multi-account channels) overrides per-channel settings.

### Per-agent heartbeats

If any `agents.list[]` entry includes a `heartbeat` block, **only those agents** run heartbeats. The per-agent block merges on top of `agents.defaults.heartbeat` (so you can set shared defaults once and override per agent).

Example: two agents, only the second agent runs heartbeats.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explicit delivery to last contact (default is "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Active hours example

Restrict heartbeats to business hours in a specific timezone:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explicit delivery to last contact (default is "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // optional; uses your userTimezone if set, otherwise host tz
        },
      },
    },
  },
}
```

Outside this window (before 9am or after 10pm Eastern), heartbeats are skipped. The next scheduled tick inside the window will run normally.

### 24/7 setup

If you want heartbeats to run all day, use one of these patterns:

- Omit `activeHours` entirely (no time-window restriction; this is the default behavior).
- Set a full-day window: `activeHours: { start: "00:00", end: "24:00" }`.

> **Note:** Do not set the same `start` and `end` time (for example `08:00` to `08:00`). That is treated as a zero-width window, so heartbeats are always skipped.


### Multi-account example

Use `accountId` to target a specific account on multi-account channels like Telegram:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // optional: route to a specific topic/thread
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Field notes


  Heartbeat interval (duration string; default unit = minutes).


  Optional model override for heartbeat runs (`provider/model`).


  When enabled, also deliver the separate `Reasoning:` message when available (same shape as `/reasoning on`).


  When true, heartbeat runs use lightweight bootstrap context and keep only `HEARTBEAT.md` from workspace bootstrap files.


  When true, each heartbeat runs in a fresh session with no prior conversation history. Uses the same isolation pattern as cron `sessionTarget: "isolated"`. Dramatically reduces per-heartbeat token cost. Combine with `lightContext: true` for maximum savings. Delivery routing still uses the main session context.


  When true, heartbeat runs defer on extra busy lanes: subagent or nested command work. Cron lanes always defer heartbeats, even without this flag, so local-model hosts do not run cron and heartbeat prompts at the same time.


  Optional session key for heartbeat runs.

- `main` (default): agent main session.
- Explicit session key (copy from `openclaw sessions --json` or the [sessions CLI](/cli/sessions)).
- Session key formats: see [Sessions](/concepts/session) and [Groups](/channels/groups).



- `last`: deliver to the last used external channel.
- explicit channel: any configured channel or plugin id, for example `discord`, `matrix`, `telegram`, or `whatsapp`.
- `none` (default): run the heartbeat but **do not deliver** externally.



  Controls direct/DM delivery behavior. `allow`: allow direct/DM heartbeat delivery. `block`: suppress direct/DM delivery (`reason=dm-blocked`).



  Optional recipient override (channel-specific id, e.g. E.164 for WhatsApp or a Telegram chat id). For Telegram topics/threads, use `<chatId>:topic:<messageThreadId>`.



  Optional account id for multi-account channels. When `target: "last"`, the account id applies to the resolved last channel if it supports accounts; otherwise it is ignored. If the account id does not match a configured account for the resolved channel, delivery is skipped.



  Overrides the default prompt body (not merged).



  Max chars allowed after `HEARTBEAT_OK` before delivery.



  When true, suppresses tool error warning payloads during heartbeat runs.



  Restricts heartbeat runs to a time window. Object with `start` (HH:MM, inclusive; use `00:00` for start-of-day), `end` (HH:MM exclusive; `24:00` allowed for end-of-day), and optional `timezone`.

- Omitted or `"user"`: uses your `agents.defaults.userTimezone` if set, otherwise falls back to the host system timezone.
- `"local"`: always uses the host system timezone.
- Any IANA identifier (e.g. `America/New_York`): used directly; if invalid, falls back to the `"user"` behavior above.
- `start` and `end` must not be equal for an active window; equal values are treated as zero-width (always outside the window).
- Outside the active window, heartbeats are skipped until the next tick inside the window.



## Delivery behavior

### Session and target routing

-
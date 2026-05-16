---
domain: channels
topic: "Group Messaging Configuration"
type: integration
keywords:
  - groups
  - group messages
  - group chat
  - allowGroups
  - group config
source: channels/groups.md
---

OpenClaw treats group chats consistently across surfaces: Discord, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo.

## Beginner intro (2 minutes)

OpenClaw "lives" on your own messaging accounts. There is no separate WhatsApp bot user. If **you** are in a group, OpenClaw can see that group and respond there.

Default behavior:

- Groups are restricted (`groupPolicy: "allowlist"`).
- Replies require a mention unless you explicitly disable mention gating.
- Normal final replies in groups/channels are private by default. Visible room output uses the `message` tool.

Translation: allowlisted senders can trigger OpenClaw by mentioning it.

> **Note:** **TL;DR**

- **DM access** is controlled by `*.allowFrom`.
- **Group access** is controlled by `*.groupPolicy` + allowlists (`*.groups`, `*.groupAllowFrom`).
- **Reply triggering** is controlled by mention gating (`requireMention`, `/activation`).



Quick flow (what happens to a group message):

```
groupPolicy? disabled -> drop
groupPolicy? allowlist -> group allowed? no -> drop
requireMention? yes -> mentioned? no -> store for context only
otherwise -> reply
```

## Visible replies

For group/channel rooms, OpenClaw defaults to `messages.groupChat.visibleReplies: "message_tool"`.
`openclaw doctor --fix` writes this default into configured-channel configs that omit it.
That means the agent still processes the turn and can update memory/session state, but its normal final answer is not automatically posted back into the room. To speak visibly, the agent uses `message(action=send)`.

This default depends on a model/runtime that reliably calls tools. If logs show
assistant text but `didSendViaMessagingTool: false`, the model answered
privately instead of calling the message tool. That is not a
Discord/Slack/Telegram send failure. Use a tool-call-reliable model for
group/channel sessions, or set
`messages.groupChat.visibleReplies: "automatic"` to restore legacy visible
final replies.

If the message tool is unavailable under the active tool policy, OpenClaw falls
back to automatic visible replies instead of silently suppressing the response.
`openclaw doctor` warns about this mismatch.

For direct chats and any other source turn, use `messages.visibleReplies: "message_tool"` to apply the same tool-only visible-reply behavior globally. Harnesses can also choose this as their unset default; the Codex harness does this for Codex-mode direct chats. `messages.groupChat.visibleReplies` remains the more specific override for group/channel rooms.

This replaces the old pattern of forcing the model to answer `NO_REPLY` for most lurk-mode turns. In tool-only mode, doing nothing visible simply means not calling the message tool.

Typing indicators are still sent while the agent works in tool-only mode. The default group typing mode is upgraded from "message" to "instant" for these turns because there may never be normal assistant message text before the agent decides whether to call the message tool. Explicit typing-mode config still wins.

To restore legacy automatic final replies for group/channel rooms:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

The gateway hot-reloads `messages` config after the file is saved. Restart only
when file watching or config reload is disabled in the deployment.

To require visible output to go through the message tool for every source chat:

```json5
{
  messages: {
    visibleReplies: "message_tool",
  },
}
```

Native slash commands (Discord, Telegram, and other surfaces with native command support) bypass `visibleReplies: "message_tool"` and always reply visibly so the channel-native command UI gets the response it expects. This applies to validated native command turns only; text-typed `/...` commands and ordinary chat turns still follow the configured group default.

## Context visibility and allowlists

Two different controls are involved in group safety:

- **Trigger authorization**: who can trigger the agent (`groupPolicy`, `groups`, `groupAllowFrom`, channel-specific allowlists).
- **Context visibility**: what supplemental context is injected into the model (reply text, quotes, thread history, forwarded metadata).

By default, OpenClaw prioritizes normal chat behavior and keeps context mostly as received. This means allowlists primarily decide who can trigger actions, not a universal redaction boundary for every quoted or historical snippet.

### Current behavior is channel-specific

- Some channels already apply sender-based filtering for supplemental context in specific paths (for example Slack thread seeding, Matrix reply/thread lookups).
    - Other channels still pass quote/reply/forward context through as received.

  ### Hardening direction (planned)

- `contextVisibility: "all"` (default) keeps current as-received behavior.
    - `contextVisibility: "allowlist"` filters supplemental context to allowlisted senders.
    - `contextVisibility: "allowlist_quote"` is `allowlist` plus one explicit quote/reply exception.

    Until this hardening model is implemented consistently across channels, expect differences by surface.

  ![Group message flow](/images/groups-flow.svg)

If you want...

| Goal                                         | What to set                                                |
| -------------------------------------------- | ---------------------------------------------------------- |
| Allow all groups but only reply on @mentions | `groups: { "*": { requireMention: true } }`                |
| Disable all group replies                    | `groupPolicy: "disabled"`                                  |
| Only specific groups                         | `groups: { "<group-id>": { ... } }` (no `"*"` key)         |
| Only you can trigger in groups               | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |
| Reuse one trusted sender set across channels | `groupAllowFrom: ["accessGroup:operators"]`                |

For reusable sender allowlists, see [Access groups](/channels/access-groups).

## Session keys

- Group sessions use `agent:<agentId>:<channel>:group:<id>` session keys (rooms/channels use `agent:<agentId>:<channel>:channel:<id>`).
- Telegram forum topics add `:topic:<threadId>` to the group id so each topic has its own session.
- Direct chats use the main session (or per-sender if configured).
- Heartbeats are skipped for group sessions.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## Pattern: personal DMs + public groups (single agent)

Yes — this works well if your "personal" traffic is **DMs** and your "public" traffic is **groups**.

Why: in single-agent mode, DMs typically land in the **main** session key (`agent:main:main`), while groups always use **non-main** session keys (`agent:main:<channel>:group:<id>`). If you enable sandboxing with `mode: "non-main"`, those group sessions run in the configured sandbox backend while your main DM session stays on-host. Docker is the default backend if you do not choose one.

This gives you one agent "brain" (shared workspace + memory), but two execution postures:

- **DMs**: full tools (host)
- **Groups**: sandbox + restricted tools

> **Note:** If you need truly separate workspaces/personas ("personal" and "public" must never mix), use a second agent + bindings. See [Multi-Agent Routing](/concepts/multi-agent).


**DMs on host, groups sandboxed:**

```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main", // groups/channels are non-main -> sandboxed
            scope: "session", // strongest isolation (one container per group/channel)
            workspaceAccess: "none",
          },
        },
      },
      tools: {
        sandbox: {
          tools: {
            // If allow is non-empty, everything else is blocked (deny still wins).
            allow: ["group:messaging", "group:sessions"],
            deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
          },
        },
      },
    }
    ```
  
**Groups see only an allowlisted folder:**

Want "groups can only see folder X" instead of "no host access"? Keep `workspaceAccess: "none"` and mount only allowlisted paths into the sandbox:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",
            scope: "session",
            workspaceAccess: "none",
            docker: {
              binds: [
                // hostPath:containerPath:mode
                "/home/user/FriendsShared:/data:ro",
              ],
            },
          },
        },
      },
    }
    ```

  
Related:

- Configuration keys and defaults: [Gateway configuration](/gateway/config-agents#agentsdefaultssandbox)
- Debugging why a tool is blocked: [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated)
- Bind mounts details: [Sandboxing](/gateway/sandboxing#custom-bind-mounts)

## Display labels

- UI labels use `displayName` when available, formatted as `<channel>:<token>`.
- `#room` is reserved for rooms/channels; group chats use `g-<slug>` (lowercase, spaces -> `-`, keep `#@+._-`).

## Group policy

Control how group/room messages are handled per channel:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // numeric Telegram user id (wizard can resolve @username)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { allow: true } } },
      },
    },
  
---
domain: tools
topic: "Reactions and Elevated Tool Access"
type: reference
keywords:
  - reactions
  - elevated
  - elevated tools
  - message reactions
  - emoji reactions
source: 
  - tools/reactions.md
  - tools/elevated.md
---

## Reactions

The agent can add and remove emoji reactions on messages using the `message`
tool with the `react` action. Reaction behavior varies by channel and transport.

## How it works

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- `emoji` is required when adding a reaction.
- Set `emoji` to an empty string (`""`) to remove the bot's reaction(s).
- Set `remove: true` to remove a specific emoji (requires non-empty `emoji`).
- On channels that support status reactions, `trackToolCalls: true` on a
  reaction lets the runtime use that reacted message for subsequent tool
  progress reactions during the same turn.

## Channel behavior

### Discord and Slack

- Empty `emoji` removes all of the bot's reactions on the message.
    - `remove: true` removes just the specified emoji.

  ### Google Chat

- Empty `emoji` removes the app's reactions on the message.
    - `remove: true` removes just the specified emoji.

  ### Telegram

- Empty `emoji` removes the bot's reactions.
    - `remove: true` also removes reactions but still requires a non-empty `emoji` for tool validation.

  ### WhatsApp

- Empty `emoji` removes the bot reaction.
    - `remove: true` maps to empty emoji internally (still requires `emoji` in the tool call).

  ### Zalo Personal (zalouser)

- Requires non-empty `emoji`.
    - `remove: true` removes that specific emoji reaction.

  ### Feishu/Lark

- Use the `feishu_reaction` tool with actions `add`, `remove`, and `list`.
    - Add/remove requires `emoji_type`; remove also requires `reaction_id`.

  ### Signal

- Inbound reaction notifications are controlled by `channels.signal.reactionNotifications`: `"off"` disables them, `"own"` (default) emits events when users react to bot messages, and `"all"` emits events for all reactions.

  ### iMessage

- Outbound reactions are iMessage tapbacks (`love`, `like`, `dislike`, `laugh`, `emphasize`, and `question`).
    - Inbound tapback notifications are controlled by `channels.imessage.reactionNotifications`: `"off"` disables them, `"own"` (default) emits events when users react to bot-authored messages, and `"all"` emits events for all tapbacks from authorized senders.

  ## Reaction level

Per-channel `reactionLevel` config controls how broadly the agent uses reactions. Values are typically `off`, `ack`, `minimal`, or `extensive`.

- [Telegram reactionLevel](/channels/telegram#reaction-notifications) — `channels.telegram.reactionLevel`
- [WhatsApp reactionLevel](/channels/whatsapp#reaction-level) — `channels.whatsapp.reactionLevel`

Set `reactionLevel` on individual channels to tune how actively the agent reacts to messages on each platform.

## Related

- [Agent Send](/tools/agent-send) — the `message` tool that includes `react`
- [Channels](/channels) — channel-specific configuration


## Elevated Tool Access

When an agent runs inside a sandbox, its `exec` commands are confined to the
sandbox environment. **Elevated mode** lets the agent break out and run commands
outside the sandbox instead, with configurable approval gates.

> **Note:** Elevated mode only changes behavior when the agent is **sandboxed**. For
  unsandboxed agents, exec already runs on the host.


## Directives

Control elevated mode per-session with slash commands:

| Directive        | What it does                                                           |
| ---------------- | ---------------------------------------------------------------------- |
| `/elevated on`   | Run outside the sandbox on the configured host path, keep approvals    |
| `/elevated ask`  | Same as `on` (alias)                                                   |
| `/elevated full` | Run outside the sandbox on the configured host path and skip approvals |
| `/elevated off`  | Return to sandbox-confined execution                                   |

Also available as `/elev on|off|ask|full`.

Send `/elevated` with no argument to see the current level.

## How it works

**Check availability**

Elevated must be enabled in config and the sender must be on the allowlist:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  
**Set the level**

Send a directive-only message to set the session default:

    ```
    /elevated full
    ```

    Or use it inline (applies to that message only):

    ```
    /elevated on run the deployment script
    ```

  
**Commands run outside the sandbox**

With elevated active, `exec` calls leave the sandbox. The effective host is
    `gateway` by default, or `node` when the configured/session exec target is
    `node`. In `full` mode, exec approvals are skipped. In `on`/`ask` mode,
    configured approval rules still apply.
  
## Resolution order

1. **Inline directive** on the message (applies only to that message)
2. **Session override** (set by sending a directive-only message)
3. **Global default** (`agents.defaults.elevatedDefault` in config)

## Availability and allowlists

- **Global gate**: `tools.elevated.enabled` (must be `true`)
- **Sender allowlist**: `tools.elevated.allowFrom` with per-channel lists
- **Per-agent gate**: `agents.list[].tools.elevated.enabled` (can only further restrict)
- **Per-agent allowlist**: `agents.list[].tools.elevated.allowFrom` (sender must match both global + per-agent)
- **Discord fallback**: if `tools.elevated.allowFrom.discord` is omitted, `channels.discord.allowFrom` is used as fallback
- **All gates must pass**; otherwise elevated is treated as unavailable

Allowlist entry formats:

| Prefix                  | Matches                         |
| ----------------------- | ------------------------------- |
| (none)                  | Sender ID, E.164, or From field |
| `name:`                 | Sender display name             |
| `username:`             | Sender username                 |
| `tag:`                  | Sender tag                      |
| `id:`, `from:`, `e164:` | Explicit identity targeting     |

## What elevated does not control

- **Tool policy**: if `exec` is denied by tool policy, elevated cannot override it.
- **Host selection policy**: elevated does not turn `auto` into a free cross-host override. It uses the configured/session exec target rules, choosing `node` only when the target is already `node`.
- **Separate from `/exec`**: the `/exec` directive adjusts per-session exec defaults for authorized senders and does not require elevated mode.

> **Note:** The bash chat command (`!` prefix; `/bash` alias) is a separate gate that requires `tools.elevated` to be enabled in addition to its own `tools.bash.enabled` flag. Disabling elevated locks `!` shell commands out as well.


## Related

**Exec tool:** Shell command execution from the agent.
  

**Exec approvals:** Approval and allowlist system for `exec`.
  

**Sandboxing:** Gateway-level sandbox configuration.
  

**Sandbox vs Tool Policy vs Elevated:** How the three gates compose during a tool call.
  


---
domain: tools
topic: "Agent Communication Tools: agent_send, Channel Messaging, and Reactions"
type: reference
keywords:
  - agent_send
  - reactions
  - channel messaging
  - outbound messages
  - emoji reactions
related:
  - concepts/messages
  - tools/tools-overview
source:
  - tools/agent-send.md
  - tools/reactions.md
---

Agent communication tools: agent_send for sending messages to channels, and reactions for adding emoji reactions.

## Agent Send

`openclaw agent` runs a single agent turn from the command line without needing
an inbound chat message. Use it for scripted workflows, testing, and
programmatic delivery.

## Quick start

    ```bash
    openclaw agent --message "What is the weather today?"
    ```

    This sends the message through the Gateway and prints the reply.

    ```bash
    # Target a specific agent
    openclaw agent --agent ops --message "Summarize logs"

    # Target a phone number (derives session key)
    openclaw agent --to +15555550123 --message "Status update"

    # Reuse an existing session
    openclaw agent --session-id abc123 --message "Continue the task"
    ```

    ```bash
    # Deliver to WhatsApp (default channel)
    openclaw agent --to +15555550123 --message "Report ready" --deliver

    # Deliver to Slack
    openclaw agent --agent ops --message "Generate report" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

## Flags

| Flag                          | Description                                                 |
| ----------------------------- | ----------------------------------------------------------- |
| `--message \<text\>`          | Message to send (required)                                  |
| `--to \<dest\>`               | Derive session key from a target (phone, chat id)           |
| `--agent \<id\>`              | Target a configured agent (uses its `main` session)         |
| `--session-id \<id\>`         | Reuse an existing session by id                             |
| `--local`                     | Force local embedded runtime (skip Gateway)                 |
| `--deliver`                   | Send the reply to a chat channel                            |
| `--channel \<name\>`          | Delivery channel (whatsapp, telegram, discord, slack, etc.) |
| `--reply-to \<target\>`       | Delivery target override                                    |
| `--reply-channel \<name\>`    | Delivery channel override                                   |
| `--reply-account \<id\>`      | Delivery account id override                                |
| `--thinking \<level\>`        | Set thinking level for the selected model profile           |
| `--verbose \<on\|full\|off\>` | Set verbose level                                           |
| `--timeout \<seconds\>`       | Override agent timeout                                      |
| `--json`                      | Output structured JSON                                      |

## Behavior

- By default, the CLI goes **through the Gateway**. Add `--local` to force the
  embedded runtime on the current machine.
- If the Gateway is unreachable, the CLI **falls back** to the local embedded run.
- Session selection: `--to` derives the session key (group/channel targets
  preserve isolation; direct chats collapse to `main`).
- Thinking and verbose flags persist into the session store.
- Output: plain text by default, or `--json` for structured payload + metadata.

## Examples

```bash
# Simple turn with JSON output
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json

# Turn with thinking level
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium

# Deliver to a different channel than the session
openclaw agent --agent ops --message "Alert" --deliver --reply-channel telegram --reply-to "@admin"
```

## Related

- [Agent CLI reference](/cli/agent)
- [Sub-agents](/tools/subagents) — background sub-agent spawning
- [Sessions](/concepts/session) — how session keys work

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

## Channel behavior

    - Empty `emoji` removes all of the bot's reactions on the message.
    - `remove: true` removes just the specified emoji.

    - Empty `emoji` removes the app's reactions on the message.
    - `remove: true` removes just the specified emoji.

    - Empty `emoji` removes the bot's reactions.
    - `remove: true` also removes reactions but still requires a non-empty `emoji` for tool validation.

    - Empty `emoji` removes the bot reaction.
    - `remove: true` maps to empty emoji internally (still requires `emoji` in the tool call).

    - Requires non-empty `emoji`.
    - `remove: true` removes that specific emoji reaction.

    - Use the `feishu_reaction` tool with actions `add`, `remove`, and `list`.
    - Add/remove requires `emoji_type`; remove also requires `reaction_id`.

    - Inbound reaction notifications are controlled by `channels.signal.reactionNotifications`: `"off"` disables them, `"own"` (default) emits events when users react to bot messages, and `"all"` emits events for all reactions.

## Reaction level

Per-channel `reactionLevel` config controls how broadly the agent uses reactions. Values are typically `off`, `ack`, `minimal`, or `extensive`.

- [Telegram reactionLevel](/channels/telegram#reaction-notifications) — `channels.telegram.reactionLevel`
- [WhatsApp reactionLevel](/channels/whatsapp#reaction-level) — `channels.whatsapp.reactionLevel`

Set `reactionLevel` on individual channels to tune how actively the agent reacts to messages on each platform.

## Related

- [Agent Send](/tools/agent-send) — the `message` tool that includes `react`
- [Channels](/channels) — channel-specific configuration

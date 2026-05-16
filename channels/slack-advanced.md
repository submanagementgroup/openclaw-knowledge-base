---
domain: channels
topic: "Slack Advanced Configuration and Mantis Integration"
type: reference
keywords:
  - Slack
  - slack advanced
  - slack Mantis
  - Slack socket mode
  - slack workspace
source: channels/slack.md
---

gs only when you need workspace- or host-specific tuning:

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

Use this only for Socket Mode workspaces that log Slack websocket pong/server-ping timeouts or run on hosts with known event-loop starvation. `clientPingTimeout` is the pong wait after the SDK sends a client ping; `serverPingTimeout` is the wait for Slack server pings. App messages and events remain application state, not transport liveness signals.

## Manifest and scope checklist

The base Slack app manifest is the same for Socket Mode and HTTP Request URLs. Only the `settings` block (and the slash command `url`) differs.

Base manifest (Socket Mode default):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

For **HTTP Request URLs mode**, replace `settings` with the HTTP variant and add `url` to each slash command. Public URL required:

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### Additional manifest settings

Surface different features that extend the above defaults.

The default manifest enables the Slack App Home **Home** tab and subscribes to `app_home_opened`. When a workspace member opens the Home tab, OpenClaw publishes a safe default Home view with `views.publish`; no conversation payload or private configuration is included. The **Messages** tab remains enabled for Slack DMs.

### Optional native slash commands

Multiple [native slash commands](#commands-and-slash-behavior) can be used instead of a single configured command with nuance:

    - Use `/agentstatus` instead of `/status` because the `/status` command is reserved.
    - No more than 25 slash commands can be made available at once.

    Replace your existing `features.slash_commands` section with a subset of [available commands](/tools/slash-commands#command-list):

    **Socket Mode (default):**

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Start a new session",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "Reset the current session"
    },
    {
      "command": "/compact",
      "description": "Compact the session context",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "Stop the current run"
    },
    {
      "command": "/session",
      "description": "Manage thread-binding expiry",
      "usage_hint": "idle <duration|off> or max-age <duration|off>"
    },
    {
      "command": "/think",
      "description": "Set the thinking level",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "Toggle verbose output",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "Show or set fast mode",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "Toggle reasoning visibility",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "Toggle elevated mode",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "Show or set exec defaults",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/model",
      "description": "Show or set the model",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "List providers/models",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "Show the short help summary"
    },
    {
      "command": "/commands",
      "description": "Show the generated command catalog"
    },
    {
      "command": "/tools",
      "description": "Show what the current agent can use right now",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "Show runtime status, including provider usage/quota when available"
    },
    {
      "command": "/tasks",
      "description": "List active/recent background tasks for the current session"
    },
    {
      "command": "/context",
      "description": "Explain how context is assembled",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "Show your sender identity"
    },
    {
      "command": "/skill",
      "description": "Run a skill by name",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "Ask a side question without changing session context",
      "usage_hint": "<question>"
    },
    {
      "command": "/side",
      "description": "Ask a side question without changing session context",
      "usage_hint": "<question>"
    },
    {
      "command": "/usage",
      "description": "Control the usage footer or show cost summary",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      
**HTTP Request URLs:**

Use the same `slash_commands` list as Socket Mode above, and add `"url": "https://gateway-host.example.com/slack/events"` to every entry. Example:

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Start a new session",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "Show the short help summary",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        Repeat that `url` value on every command in the list.

      
### Optional authorship scopes (write operations)

Add the `chat:write.customize` bot scope if you want outgoing messages to use the active agent identity (custom username and icon) instead of the default Slack app identity.

    If you use an emoji icon, Slack expects `:emoji_name:` syntax.

  ### Optional user-token scopes (read operations)

If you configure `channels.slack.userToken`, typical read scopes are:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (if you depend on Slack search reads)

  ## Token model

- `botToken` + `appToken` are required for Socket Mode.
- HTTP mode requires `botToken` + `signingSecret`.
- `botToken`, `appToken`, `signingSecret`, and `userToken` accept plaintext
  strings or SecretRef objects.
- Config tokens override env fallback.
- `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` env fallback applies only to the default account.
- `userToken` (`xoxp-...`) is config-only (no env fallback) and defaults to read-only behavior (`userTokenReadOnly: true`).

Status snapshot behavior:

- Slack account inspection tracks per-credential `*Source` and `*Status`
  fields (`botToken`, `appToken`, `signingSecret`, `userToken`).
- Status is `available`, `configured_unavailable`, or `missing`.
- `configured_unavailable` means the account is configured through SecretRef
  or another non-inline secret source, but the current command/runtime path
  could not resolve the actual value.
- In HTTP mode, `signingSecretStatus` is included; in Socket Mode, the
  required pair is `botTokenStatus` + `appTokenStatus`.

> **Note:** For actions/directory reads, user token can be preferred when configured. For writes, bot token remains preferred; user-token writes are only allowed when `userTokenReadOnly: false` and bot token is unavailable.


## Actions and gates

Slack actions are controlled by `channels.slack.actions.*`.

Available action groups in current Slack tooling:

| Group      | Default |
| ---------- | ------- |
| messages   | enabled |
| reactions  | enabled |
| pins       | enabled |
| memberInfo | enabled |
| emojiList  | enabled |

Current Slack message actions include `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info`, and `emoji-list`. `download-file` accepts Slack file IDs shown in inbound file placeholders and returns image previews for images or local file metadata for other file types.

## Access control and routing

**DM policy:**

`channels.slack.dmPolicy` controls DM access. `channels.slack.allowFrom` is the canonical DM allowlist.

    - `pairing` (default)
    - `allowlist`
    - `open` (requires `channels.slack.allowFrom` to include `"*"`)
    - `disabled`

    DM flags:

    - `dm.enabled` (default true)
    - `channels.slack.allowFrom`
    - `dm.allowFrom` (legacy)
    - `dm.groupEnabled` (group DMs default false)
    - `dm.groupChannels` (optional MPIM allowlist)

    Multi-account precedence:

    - `channels.slack.accounts.default.allowFrom` applies only to the `default` account.
    - Named accounts inherit `channels.slack.allowFrom` when their own `allowFrom` is unset.
    - Named accounts do not inherit `channels.slack.accounts.default.allowFrom`.

    Legacy `channels.slack.dm.policy` and `channels.slack.dm.allowFrom` still read for compatibility. `openclaw doctor --fix` migrates them to `dmPolicy` and `allowFrom` when it can do so without changing access.

    Pairing in DMs uses `openclaw pairing approve slack <code>`.

  
**Channel policy:**

`channels.slack.groupPolicy` controls channel handling:

    - `open`
    - `allowlist`
    - `disabled`

    Channel allowlist lives under `channels.slack.channels` and **must use stable Slack channel IDs** (for example `C12345678`) as config keys.

    Runtime note: if `channels.slack` is completely missing (env-only setup), runtime falls back to `groupPolicy="allowlist"` and logs a warning (even if `channels.defaults.groupPolicy` is set).

    Name/ID resolution:

    - channel allowlist entries and DM allowlist entries are resolved at startup when token access allows
    - unresolved channel-name entries are kept as configured but ignored for routing by default
    - inbound authorization and channel routing are ID-first by default; direct username/slug matching requires `channels.slack.dangerouslyAllowNameMatching: true`

    > **Note:** Name-based keys (`#channel-name` or `channel-name`) do **not** match under `groupPolicy: "allowlist"`. The channel lookup is ID-first by default, so a name-based key will never route successfully and all messages in that channel will be silently blocked. This differs from `groupPolicy: "open"`, where the channel key is not required for routing and a name-based key appears to work.

    Always use the Slack channel ID as the key. To find it: right-click the channel in Slack → **Copy link** — the ID (`C...`) appears at the end of the URL.

    Correct:

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { allow: true, requireMention: true },
          },
        },
      },
    }
    ```

    Incorrect (silently blocked under `groupPolicy: "allowlist"`):

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { allow: true, requireMention: true },
          },
        },
      },
    }
    ```
    

  
**Mentions and channel users:**

Channel messages are mention-gated by default.

    Mention sources:

    - explicit app mention (`<@botId>`)
    - Slack user-group mention (`<!subteam^S...>`) when the bo
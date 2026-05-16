---
domain: channels
topic: "Slack Channel Setup and Configuration"
type: procedure
keywords:
  - Slack
  - slack bot
  - slack setup
  - Slack channel
  - SLACK_BOT_TOKEN
  - slack config
  - slack app
source: channels/slack.md
---

Production-ready for DMs and channels via Slack app integrations. Default mode is Socket Mode; HTTP Request URLs are also supported.

**Pairing:** Slack DMs default to pairing mode.
  

**Slash commands:** Native command behavior and command catalog.
  

**Channel troubleshooting:** Cross-channel diagnostics and repair playbooks.
  

## Choosing Socket Mode or HTTP Request URLs

Both transports are production-ready and reach feature parity for messaging, slash commands, App Home, and interactivity. Pick by deployment shape, not features.

| Concern                      | Socket Mode (default)                                                                | HTTP Request URLs                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Public Gateway URL           | Not required                                                                         | Required (DNS, TLS, reverse proxy or tunnel)                                                                   |
| Outbound network             | Outbound WSS to `wss-primary.slack.com` must be reachable                            | No outbound WS; inbound HTTPS only                                                                             |
| Tokens needed                | Bot token (`xoxb-...`) + App-Level Token (`xapp-...`) with `connections:write`       | Bot token (`xoxb-...`) + Signing Secret                                                                        |
| Dev laptop / behind firewall | Works as-is                                                                          | Needs a public tunnel (ngrok, Cloudflare Tunnel, Tailscale Funnel) or staging Gateway                          |
| Horizontal scaling           | One Socket Mode session per app per host; multiple Gateways need separate Slack apps | Stateless POST handler; multiple Gateway replicas can share one app behind a load balancer                     |
| Multi-account on one Gateway | Supported; each account opens its own WS                                             | Supported; each account needs a unique `webhookPath` (default `/slack/events`) so registrations do not collide |
| Slash command transport      | Delivered over the WS connection; `slash_commands[].url` is ignored                  | Slack POSTs to `slash_commands[].url`; field is required for the command to dispatch                           |
| Request signing              | Not used (auth is the App-Level Token)                                               | Slack signs every request; OpenClaw verifies with `signingSecret`                                              |
| Recovery on connection drop  | Slack SDK auto-reconnects; the gateway's pong-timeout transport tuning applies       | No persistent connection to drop; retries are per-request from Slack                                           |

> **Note:** **Pick Socket Mode** for single-Gateway hosts, dev laptops, and on-prem networks that can reach `*.slack.com` outbound but cannot accept inbound HTTPS.

**Pick HTTP Request URLs** when running multiple Gateway replicas behind a load balancer, when outbound WSS is blocked but inbound HTTPS is allowed, or when you already terminate Slack webhooks at a reverse proxy.


## Install

Install Slack before configuring the channel:

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` registers and enables the plugin. The plugin still does nothing until you configure the Slack app and channel settings below. See [Plugins](/tools/plugin) for general plugin behavior and install rules.

## Quick setup

**Socket Mode (default):**

**Create a new Slack app**

Open [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → select your workspace → paste one of the manifests below → **Next** → **Create**.

        

```json Recommended
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

```json Minimal
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
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
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
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        

        > **Note:** **Recommended** matches the Slack plugin's full feature set: App Home, slash commands, files, reactions, pins, group DMs, and emoji/usergroup reads. Pick **Minimal** when workspace policy restricts scopes — it covers DMs, channel/group history, mentions, and slash commands but drops files, reactions, pins, group-DM (`mpim:*`), `emoji:read`, and `usergroups:read`. See [Manifest and scope checklist](#manifest-and-scope-checklist) for per-scope rationale and additive options like extra slash commands.
        

        After Slack creates the app:

        - **Basic Information → App-Level Tokens → Generate Token and Scopes**: add `connections:write`, save, copy the `xapp-...` value.
        - **Install App → Install to Workspace**: copy the `xoxb-...` Bot User OAuth Token.

      
**Configure OpenClaw**

Recommended SecretRef setup:

```bash
export SLACK_APP_TOKEN=xapp-...
export SLACK_BOT_TOKEN=xoxb-...
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        Env fallback (default account only):

```bash
SLACK_APP_TOKEN=xapp-...
SLACK_BOT_TOKEN=xoxb-...
```

      
**Start gateway**

```bash
openclaw gateway
```

      
**HTTP Request URLs:**

**Create a new Slack app**

Open [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → select your workspace → paste one of the manifests below → replace `https://gateway-host.example.com/slack/events` with your public Gateway URL → **Next** → **Create**.

        

```json Recommended
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
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
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

```json Minimal
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
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
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
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im"
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

        

        > **Note:** **Recommended** matches the Slack plugin's full feature set; **Minimal** drops files, reactions, pins, group-DM (`mpim:*`), `emoji:read`, and `usergroups:read` for restrictive workspaces. See [Manifest and scope checklist](#manifest-and-scope-checklist) for per-scope rationale.
        

        > **Note:** The three URL fields (`slash_commands[].url`, `event_subscriptions.request_url`, and `interactivity.request_url` / `message_menu_options_url`) all point at the same OpenClaw endpoint. Slack's manifest schema requires them named separately, but OpenClaw routes by payload type so a single `webhookPath` (default `/slack/events`) is enough. Slash commands without `slash_commands[].url` will silently no-op in HTTP mode.
        

        After Slack creates the app:

        - **Basic Information → App Credentials**: copy the **Signing Secret** for request verification.
        - **Install App → Install to Workspace**: copy the `xoxb-...` Bot User OAuth Token.

      
**Configure OpenClaw**

Recommended SecretRef setup:

```bash
export SLACK_BOT_TOKEN=xoxb-...
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        > **Note:** Use unique webhook paths for multi-account HTTP

        Give each account a distinct `webhookPath` (default `/slack/events`) so registrations do not collide.
        

      
**Start gateway**

```bash
openclaw gateway
```

      
## Socket Mode transport tuning

OpenClaw sets the Slack SDK client pong timeout to 15 seconds by default for Socket Mode. Override the transport settin
---
domain: channels
topic: "iMessage and BlueBubbles Channels: macOS Messages and Cross-Platform iMessage Setup"
type: procedure
keywords:
  - iMessage
  - BlueBubbles
  - Apple Messages
  - macOS iMessage
  - cross-platform iMessage
related:
  - channels/whatsapp
  - platforms/macos-app
source:
  - channels/imessage.md
  - channels/bluebubbles.md
---

iMessage and BlueBubbles channels for OpenClaw. iMessage uses Apple Messages on macOS. BlueBubbles uses a self-hosted server for cross-platform iMessage access.

## iMessage (macOS Only)

For new iMessage deployments, use <a href="/channels/bluebubbles">BlueBubbles</a>.

The `imsg` integration is legacy and may be removed in a future release.

Status: legacy external CLI integration. Gateway spawns `imsg rpc` and communicates over JSON-RPC on stdio (no separate daemon/port).

    Preferred iMessage path for new setups.

    iMessage DMs default to pairing mode.

    Full iMessage field reference.

## Quick setup

```bash
brew install steipete/tap/imsg
imsg rpc --help
```

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

```bash
openclaw gateway
```

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        Pairing requests expire after 1 hour.

    OpenClaw only requires a stdio-compatible `cliPath`, so you can point `cliPath` at a wrapper script that SSHes to a remote Mac and runs `imsg`.

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

    Recommended config when attachments are enabled:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // used for SCP attachment fetches
      includeAttachments: true,
      // Optional: override allowed attachment roots.
      // Defaults include /Users/*/Library/Messages/Attachments
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    If `remoteHost` is not set, OpenClaw attempts to auto-detect it by parsing the SSH wrapper script.
    `remoteHost` must be `host` or `user@host` (no spaces or SSH options).
    OpenClaw uses strict host-key checking for SCP, so the relay host key must already exist in `~/.ssh/known_hosts`.
    Attachment paths are validated against allowed roots (`attachmentRoots` / `remoteAttachmentRoots`).

## Requirements and permissions (macOS)

- Messages must be signed in on the Mac running `imsg`.
- Full Disk Access is required for the process context running OpenClaw/`imsg` (Messages DB access).
- Automation permission is required to send messages through Messages.app.

Permissions are granted per process context. If gateway runs headless (LaunchAgent/SSH), run a one-time interactive command in that same context to trigger prompts:

```bash
imsg chats --limit 1
# or
imsg send <handle> "test"
```

## Access control and routing

    `channels.imessage.dmPolicy` controls direct messages:

    - `pairing` (default)
    - `allowlist`
    - `open` (requires `allowFrom` to include `"*"`)
    - `disabled`

    Allowlist field: `channels.imessage.allowFrom`.

    Allowlist entries can be handles or chat targets (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`).

    `channels.imessage.groupPolicy` controls group handling:

    - `allowlist` (default when configured)
    - `open`
    - `disabled`

    Group sender allowlist: `channels.imessage.groupAllowFrom`.

    Runtime fallback: if `groupAllowFrom` is unset, iMessage group sender checks fall back to `allowFrom` when available.
    Runtime note: if `channels.imessage` is completely missing, runtime falls back to `groupPolicy="allowlist"` and logs a warning (even if `channels.defaults.groupPolicy` is set).

    Mention gating for groups:

    - iMessage has no native mention metadata
    - mention detection uses regex patterns (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - with no configured patterns, mention gating cannot be enforced

    Control commands from authorized senders can bypass mention gating in groups.

    - DMs use direct routing; groups use group routing.
    - With default `session.dmScope=main`, iMessage DMs collapse into the agent main session.
    - Group sessions are isolated (`agent:<agentId>:imessage:group:<chat_id>`).
    - Replies route back to iMessage us

## BlueBubbles (Cross-Platform iMessage)

Status: bundled plugin that talks to the BlueBubbles macOS server over HTTP. **Recommended for iMessage integration** due to its richer API and easier setup compared to the legacy imsg channel.

Current OpenClaw releases bundle BlueBubbles, so normal packaged builds do not need a separate `openclaw plugins install` step.

## Overview

- Runs on macOS via the BlueBubbles helper app ([bluebubbles.app](https://bluebubbles.app)).
- Recommended/tested: macOS Sequoia (15). macOS Tahoe (26) works; edit is currently broken on Tahoe, and group icon updates may report success but not sync.
- OpenClaw talks to it through its REST API (`GET /api/v1/ping`, `POST /message/text`, `POST /chat/:id/*`).
- Incoming messages arrive via webhooks; outgoing replies, typing indicators, read receipts, and tapbacks are REST calls.
- Attachments and stickers are ingested as inbound media (and surfaced to the agent when possible).
- Auto-TTS replies that synthesize MP3 or CAF audio are delivered as iMessage voice memo bubbles instead of plain file attachments.
- Pairing/allowlist works the same way as other channels (`/channels/pairing` etc) with `channels.bluebubbles.allowFrom` + pairing codes.
- Reactions are surfaced as system events just like Slack/Telegram so agents can "mention" them before replying.
- Advanced features: edit, unsend, reply threading, message effects, group management.

## Quick start

    Install the BlueBubbles server on your Mac (follow the instructions at [bluebubbles.app/install](https://bluebubbles.app/install)).

    In the BlueBubbles config, enable the web API and set a password.

    Run `openclaw onboard` and select BlueBubbles, or configure manually:

    ```json5
    {
      channels: {
        bluebubbles: {
          enabled: true,
          serverUrl: "http://192.168.1.100:1234",
          password: "example-password",
          webhookPath: "/bluebubbles-webhook",
        },
      },
    }
    ```

    Point BlueBubbles webhooks to your gateway (example: `https://your-gateway-host:3000/bluebubbles-webhook?password=<password>`).

    Start the gateway; it will register the webhook handler and start pairing.

**Security**

- Always set a webhook password.
- Webhook authentication is always required. OpenClaw rejects BlueBubbles webhook requests unless they include a password/guid that matches `channels.bluebubbles.password` (for example `?password=<password>` or `x-password`), regardless of loopback/proxy topology.
- Password authentication is checked before reading/parsing full webhook bodies.

## Keeping Messages.app alive (VM / headless setups)

Some macOS VM / always-on setups can end up with Messages.app going "idle" (incoming events stop until the app is opened/foregrounded). A simple workaround is to **poke Messages every 5 minutes** using an AppleScript + LaunchAgent.

    Save this as `~/Scripts/poke-messages.scpt`:

    ```applescript
    try
      tell application "Messages"
        if not running then
          launch
        end if

        -- Touch the scripting interface to keep the process responsive.
        set _chatCount to (count of chats)
      end tell
    on error
      -- Ignore transient failures (first-run prompts, locked session, etc).
    end try
    ```

    Save this as `~/Library/LaunchAgents/com.user.poke-messages.plist`:

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
      <dict>
        <key>Label</key>
        <string>com.user.poke-messages</string>

        <key>ProgramArguments</key>
        <array>
          <string>/bin/bash</string>
          <string>-lc</string>
          <string>/usr/bin/osascript &quot;$HOME/Scripts/poke-messages.scpt&quot;</string>
        </array>

        <key>RunAtLoad</key>
        <true/>

        <key>StartInterval</key>
        <integer>300</integer>

        <key>StandardOutPath</key>
        <string>/tmp/poke-messages.log</string>
        <key>StandardErrorPath</key>
        <string>/tmp/poke-messages.err</string>
      </dict>
    </plist>
    ```

    This runs **every 300 seconds** and **on login**. The first run may trigger macOS **Automation** prompts (`osascript` → Messages). Approve them in the same user session that runs the LaunchAgent.

    ```bash
    launchctl unload ~/Library/LaunchAgents/com.user.poke-messages.plist 2>/dev/null || true
    launchctl load ~/Library/LaunchAgents/com.user.poke-messages.plist
    ```

## Onboarding

BlueBubbles is available in interactive onboarding:

```
openclaw onboard
```

The wizard prompts for:

<ParamField path="Server URL" type="string" required>
  BlueBubbles server address (e.g., `http://192.168.1.100:1234`).
</ParamField>
<ParamField path="Password" type="string" required>
  API password from BlueBubbles Server settings.
</ParamField>
<ParamField path="Webhook path" type="string" default="/bluebubbles-webhook">
  Webhook endpoint path.

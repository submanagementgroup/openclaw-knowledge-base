---
domain: channels
topic: "iMessage Channel Setup via BlueBubbles"
type: procedure
keywords:
  - iMessage
  - BlueBubbles
  - imessage bot
  - iMessage channel
  - Apple Messages
source: ['channels/imessage.md', 'channels/bluebubbles.md', 'channels/imessage-from-bluebubbles.md']
---

> **Note:** For OpenClaw iMessage deployments, use `imsg` on a signed-in macOS Messages host. If your Gateway runs on Linux or Windows, point `channels.imessage.cliPath` at an SSH wrapper that runs `imsg` on the Mac.

**Gateway-downtime catchup is opt-in.** When enabled (`channels.imessage.catchup.enabled: true`), the gateway replays inbound messages that landed in `chat.db` while it was offline (crash, restart, Mac sleep) on next startup. Disabled by default — see [Catching up after gateway downtime](#catching-up-after-gateway-downtime). Closes [openclaw#78649](https://github.com/openclaw/openclaw/issues/78649).


> **Note:** BlueBubbles support was removed. Migrate `channels.bluebubbles` configs to `channels.imessage`; OpenClaw supports iMessage through `imsg` only. Start with [BlueBubbles removal and the imsg iMessage path](/announcements/bluebubbles-imessage) for the short announcement, or [Coming from BlueBubbles](/channels/imessage-from-bluebubbles) for the full migration table.


Status: native external CLI integration. Gateway spawns `imsg rpc` and communicates over JSON-RPC on stdio (no separate daemon/port). Advanced actions require `imsg launch` and a successful private API probe.

**Private API actions:** Replies, tapbacks, effects, attachments, and group management.
  

**Pairing:** iMessage DMs default to pairing mode.
  

**Remote Mac:** Use an SSH wrapper when the Gateway is not running on the Messages Mac.
  

**Configuration reference:** Full iMessage field reference.
  

## Quick setup

**Local Mac (fast path):**

**Install and verify imsg**

```bash
brew install steipete/tap/imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

      
**Configure OpenClaw**

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

      
**Start gateway**

```bash
openclaw gateway
```

      
**Approve first DM pairing (default dmPolicy)**

```bash
openclaw pairing list imessage
openclaw pairing approve imessage 
```

        Pairing requests expire after 1 hour.
      
**Remote Mac over SSH:**

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
- For advanced actions (react / edit / unsend / threaded reply / effects / group ops), System Integrity Protection must be disabled — see [Enabling the imsg private API](#enabling-the-imsg-private-api) below. Basic text and media send/receive work without it.

> **Note:** Permissions are granted per process context. If gateway runs headless (LaunchAgent/SSH), run a one-time interactive command in that same context to trigger prompts:

```bash
imsg chats --limit 1
# or
imsg send <handle> "test"
```



## Enabling the imsg private API

`imsg` ships in two operational modes:

- **Basic mode** (default, no SIP changes needed): outbound text and media via `send`, inbound watch/history, chat list. This is what you get out of the box from a fresh `brew install steipete/tap/imsg` plus the standard macOS permissions above.
- **Private API mode**: `imsg` injects a helper dylib into `Messages.app` to call internal `IMCore` functions. This is what unlocks `react`, `edit`, `unsend`, `reply` (threaded), `sendWithEffect`, `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, plus typing indicators and read receipts.

To reach the advanced action surface that this channel page documents, you need Private API mode. The `imsg` README is explicit about the requirement:

> Advanced features such as `read`, `typing`, `launch`, bridge-backed rich send, message mutation, and chat management are opt-in. They require SIP to be disabled and a helper dylib to be injected into `Messages.app`. `imsg launch` refuses to inject when SIP is enabled.

The helper-injection technique uses `imsg`'s own dylib to reach Messages private APIs. There is no third-party server or BlueBubbles runtime in the OpenClaw iMessage path.

> **Note:** **Disabling SIP is a real security tradeoff.** SIP is one of macOS's core protections against running modified system code; turning it off system-wide opens up additional attack surface and side effects. Notably, **disabling SIP on Apple Silicon Macs also disables the ability to install and run iOS apps on your Mac**.

Treat this as a deliberate operational choice, not a default. If your threat model can't tolerate SIP being off, bundled iMessage is limited to basic mode — text and media send/receive only, no reactions / edit / unsend / effects / group ops.


### Setup

1. **Install (or upgrade) `imsg`** on the Mac that runs Messages.app:

   ```bash
   brew install steipete/tap/imsg
   imsg --version
   imsg status --json
   ```

   The `imsg status --json` output reports `bridge_version`, `rpc_methods`, and per-method `selectors` so you can see what the current build supports before you start.

2. **Disable System Integrity Protection.** This is macOS-version-specific because the underlying Apple requirement depends on the OS and hardware:
   - **macOS 10.13–10.15 (Sierra–Catalina):** disable Library Validation via Terminal, reboot to Recovery Mode, run `csrutil disable`, restart.
   - **macOS 11+ (Big Sur and later), Intel:** Recovery Mode (or Internet Recovery), `csrutil disable`, restart.
   - **macOS 11+, Apple Silicon:** power-button startup sequence to enter Recovery; on recent macOS versions hold the **Left Shift** key when you click Continue, then `csrutil disable`. Virtual-machine setups follow a separate flow — take a VM snapshot first.
   - **macOS 26 / Tahoe:** library-validation policies and `image

## BlueBubbles Setup

Status: bundled plugin that talks to the BlueBubbles macOS server over HTTP. **Recommended for iMessage integration** due to its richer API and easier setup compared to the legacy imsg channel.

> **Note:** Current OpenClaw releases bundle BlueBubbles, so normal packaged builds do not need a separate `openclaw plugins install` step.


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

**Install BlueBubbles**

Install the BlueBubbles server on your Mac (follow the instructions at [bluebubbles.app/install](https://bluebubbles.app/install)).
  
**Enable the web API**

In the BlueBubbles config, enable the web API and set a password.
  
**Configure OpenClaw**

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

  
**Point webhooks at the gateway**

Point BlueBubbles webhooks to your gateway (example: `https://your-gateway-host:3000/bluebubbles-webhook?password=<password>`).
  
**Start the gateway**

Start the gateway; it will register the webhook handler and start pairing.
  
> **Note:** **Security**

- Always set a webhook password.
- Webhook authentication is always required. OpenClaw rejects BlueBubbles webhook requests unless they include a password/guid that matches `channels.bluebubbles.password` (for example `?password=<password>` or `x-password`), regardless of loopback/proxy topology.
- Password authentication is checked before reading/parsing full webhook bodies.



## Keeping Messages.app alive (VM / headless setups)

Some macOS VM / always-on setups can end up with Messages.app going "idle" (incoming events stop until the app is opened/foregrounded). A simple workaround is to **poke Messages every 5 minutes** using an AppleScript + LaunchAgent.

**Save the AppleScript**

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

  
**Install a LaunchAgent**

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

  
**Load it**

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


  BlueBubbles server address (e.g., `http://192.168.1.100:1234`).


  API password from BlueBubbles Server settings.


  Webhook endpoint path.


  `pairing`, `allowlist`, `open`, or `disabled`.


  Phone numbers, emails, or chat targets.


You can also add BlueBubbles via CLI:

```
openclaw channels add bluebubbles --http-url http://192.168.1.100:1234 --password <password>
```

## Access control (DMs + groups)

**DMs:**

- Default: `channels.bluebubbles.dmPolicy = "pairing"`.
    - Unknown senders receive a pairing code; messages are ignored until approved (codes expire after 1 hour).
    - Approve via:
      - `openclaw pairing list bluebubbles`
      - `openclaw pairing approve bluebubbles `
    - Pairing is the default token exchange. Details: [Pairing](/channels/pairing)

  
**Groups:**

- `channels.bluebubbles.groupPolicy = open | allowlist | disabled` (default: `allowlist`).
    - `channels.bluebubbles.groupAllowFrom` controls who can trigger in groups when `allowlist` is set.

  
### Contact name enrichment (macOS, optional)

BlueBubbles group webhooks often only include raw participant addresses. If you want `GroupMembers` context to show local contact names instead, you can opt in to local Contacts enrichment on macOS:

- `channels.bluebubbles.enrichGroupParticipantsFromContacts = true` enables the lookup. Default: `false`.
- Lookups run only after group access, command authorization, and mention gating have allowed the message through.
- Only unnamed phone participants are enriched.
- Raw phone numbers remain as the fallback when no local match is found.

```json5
{
  channels: {
    bluebubbles: {
      enrichGroupParticipantsFromContacts: true,
    },
  },
}
```

### Mention gating (groups)

BlueBubbles supports mention gating for group chats, matching iMessage/WhatsApp behavior:

- Uses `agents.list[].groupChat.mentionPatterns` (or `messages.groupChat.mentionPatterns`) to detect mentions.
- When `requireMention` is enabled for a group, the agent only responds when mentioned.
- Control commands from authorized senders bypass mention gating.

Per-group configuration:

```json5
{
  channels: 
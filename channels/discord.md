---
domain: channels
topic: "Discord Channel Setup: Bot Creation, Guild Config, Access Control, Slash Commands, and Voice"
type: procedure
keywords:
  - Discord
  - Discord bot
  - discord token
  - guild
  - slash commands
  - Discord channel
  - Discord voice
related:
  - channels/slack
  - channels/telegram
  - gateway/configuration-overview
source: channels/discord.md
---

Discord channel for OpenClaw. Supports DMs, server channels, threads, forum channels, interactive components, voice, and native slash commands.

## Quick Setup

1. Create a Discord bot at https://discord.com/developers/applications
2. Add the bot to your server (OAuth2 URL with `bot` + `applications.commands` scopes)
3. Configure:
   ```json5
   {
     channels: {
       discord: {
         token: "YOUR_BOT_TOKEN",
         guildId: "YOUR_GUILD_ID",
       }
     }
   }
   ```

## Key Features

Ready for DMs and guild channels via the official Discord gateway.

    Discord DMs default to pairing mode.

    Native command behavior and command catalog.

    Cross-channel diagnostics and repair flow.

## Quick setup

You will need to create a new application with a bot, add the bot to your server, and pair it to OpenClaw. We recommend adding your bot to your own private server. If you don't have one yet, [create one first](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (choose **Create My Own > For me and my friends**).

    Go to the [Discord Developer Portal](https://discord.com/developers/applications) and click **New Application**. Name it something like "OpenClaw".

    Click **Bot** on the sidebar. Set the **Username** to whatever you call your OpenClaw agent.

    Still on the **Bot** page, scroll down to **Privileged Gateway Intents** and enable:

    - **Message Content Intent** (required)
    - **Server Members Intent** (recommended; required for role allowlists and name-to-ID matching)
    - **Presence Intent** (optional; only needed for presence updates)

    Scroll back up on the **Bot** page and click **Reset Token**.

    Despite the name, this generates your first token — nothing is being "reset."

    Copy the token and save it somewhere. This is your **Bot Token** and you will need it shortly.

    Click **OAuth2** on the sidebar. You'll generate an invite URL with the right permissions to add the bot to your server.

    Scroll down to **OAuth2 URL Generator** and enable:

    - `bot`
    - `applications.commands`

    A **Bot Permissions** section will appear below. Enable at least:

    **General Permissions**
      - View Channels
    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions (optional)

    This is the baseline set for normal text channels. If you plan to post in Discord threads, including forum or media channel workflows that create or continue a thread, also enable **Send Messages in Threads**.
    Copy the generated URL at the bottom, paste it into your browser, select your server, and click **Continue** to connect. You should now see your bot in the Discord server.

    Back in the Discord app, you need to enable Developer Mode so you can copy internal IDs.

    1. Click **User Settings** (gear icon next to your avatar) → **Advanced** → toggle on **Developer Mode**
    2. Right-click your **server icon** in the sidebar → **Copy Server ID**
    3. Right-click your **own avatar** → **Copy User ID**

    Save your **Server ID** and **User ID** alongside your Bot Token — you'll send all three to OpenClaw in the next step.

    For pairing to work, Discord needs to allow your bot to DM you. Right-click your **server icon** → **Privacy Settings** → toggle on **Direct Messages**.

    This lets server members (including bots) send you DMs. Keep this enabled if you want to use Discord DMs with OpenClaw. If you only plan to use guild channels, you can disable DMs after pairing.

    Your Discord bot token is a secret (like a password). Set it on the machine running OpenClaw before messaging your agent.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    If OpenClaw is already running as a background service, restart it via the OpenClaw Mac app or by stopping and restarting the `openclaw gateway run` process.
    For managed service installs, run `openclaw gateway install` from a shell where `DISCORD_BOT_TOKEN` is present, or store the variable in `~/.openclaw/.env`, so the service can resolve the env SecretRef after restart.
    If your host is blocked or rate-limited by Discord's startup application lookup, set the Discord application/client ID from the Developer Portal so startup can skip that REST call. Use `channels.discord.applicationId` for the default account, or `channels.discord.accounts.<accountId>.applicationId` when you run multiple Discord bots.

        Chat with your OpenClaw agent on any existing channel (e.g. Telegram) and tell it. If Discord is your first channel, use the CLI / config tab instead.

        > "I already set my Discord bot token in config. Please finish Discord setup with User ID `<user_id>` and Server ID `<server_id>`."

        If you prefer file-based config, set:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Env fallback for the default account:

```bash
DISCORD_BOT_TOKEN=...
```

        For scripted or remote setup, write the same JSON5 block with `open

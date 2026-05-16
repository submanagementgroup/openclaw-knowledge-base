---
domain: channels
topic: "Discord Channel Setup and Configuration"
type: procedure
keywords:
  - Discord
  - discord bot
  - discord setup
  - Discord channel
  - DISCORD_TOKEN
  - discord config
source: channels/discord.md
---

Ready for DMs and guild channels via the official Discord gateway.

**Pairing:** Discord DMs default to pairing mode.
  

**Slash commands:** Native command behavior and command catalog.
  

**Channel troubleshooting:** Cross-channel diagnostics and repair flow.
  

## Quick setup

You will need to create a new application with a bot, add the bot to your server, and pair it to OpenClaw. We recommend adding your bot to your own private server. If you don't have one yet, [create one first](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (choose **Create My Own > For me and my friends**).

**Create a Discord application and bot**

Go to the [Discord Developer Portal](https://discord.com/developers/applications) and click **New Application**. Name it something like "OpenClaw".

    Click **Bot** on the sidebar. Set the **Username** to whatever you call your OpenClaw agent.

  
**Enable privileged intents**

Still on the **Bot** page, scroll down to **Privileged Gateway Intents** and enable:

    - **Message Content Intent** (required)
    - **Server Members Intent** (recommended; required for role allowlists and name-to-ID matching)
    - **Presence Intent** (optional; only needed for presence updates)

  
**Copy your bot token**

Scroll back up on the **Bot** page and click **Reset Token**.

    > **Note:** Despite the name, this generates your first token — nothing is being "reset."
    

    Copy the token and save it somewhere. This is your **Bot Token** and you will need it shortly.

  
**Generate an invite URL and add the bot to your server**

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

  
**Enable Developer Mode and collect your IDs**

Back in the Discord app, you need to enable Developer Mode so you can copy internal IDs.

    1. Click **User Settings** (gear icon next to your avatar) → **Advanced** → toggle on **Developer Mode**
    2. Right-click your **server icon** in the sidebar → **Copy Server ID**
    3. Right-click your **own avatar** → **Copy User ID**

    Save your **Server ID** and **User ID** alongside your Bot Token — you'll send all three to OpenClaw in the next step.

  
**Allow DMs from server members**

For pairing to work, Discord needs to allow your bot to DM you. Right-click your **server icon** → **Privacy Settings** → toggle on **Direct Messages**.

    This lets server members (including bots) send you DMs. Keep this enabled if you want to use Discord DMs with OpenClaw. If you only plan to use guild channels, you can disable DMs after pairing.

  
**Set your bot token securely (do not send it in chat)**

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

  
**Configure OpenClaw and pair**

**Ask your agent:**

Chat with your OpenClaw agent on any existing channel (e.g. Telegram) and tell it. If Discord is your first channel, use the CLI / config tab instead.

        > "I already set my Discord bot token in config. Please finish Discord setup with User ID `<user_id>` and Server ID `<server_id>`."
      
**CLI / config:**

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

        For scripted or remote setup, write the same JSON5 block with `openclaw config patch --file ./discord.patch.json5 --dry-run` and then rerun without `--dry-run`. Plaintext `token` values are supported. SecretRef values are also supported for `channels.discord.token` across env/file/exec providers. See [Secrets Management](/gateway/secrets).

        For multiple Discord bots, keep each bot token and application ID under its account. A top-level `channels.discord.applicationId` is inherited by accounts, so only set it there when every account should use the same application ID.

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      

**Approve first DM pairing**

Wait until the gateway is running, then DM your bot in Discord. It will respond with a pairing code.

    **Ask your agent:**

Send the pairing code to your agent on your existing channel:

        > "Approve this Discord pairing code: ``"
      
**CLI:**

```bash
openclaw pairing list discord
openclaw pairing approve discord 
```

      
Pairing codes expire after 1 hour.

    You should now be able to chat with your agent in Discord via DM.

  
> **Note:** Token resolution is account-aware. Config token values win over env fallback. `DISCORD_BOT_TOKEN` is only used for the default account.
If two enabled Discord accounts resolve to the same bot token, OpenClaw starts only one gateway monitor for that token. A config-sourced token wins over the default env fallback; otherwise the first enabled account wins and the duplicate account is reported disabled.
For advanced outbound calls (message tool/channel actions), an explicit per-call `token` is used for that call. This applies to send and read/probe-style actions (for example read/search/fetch/thread/pins/permissions). Account policy/retry settings still come from the selected account in the active runtime snapshot.


## Recommended: Set up a guild workspace

Once DMs are working, you can set up your Discord server as a full workspace where each channel gets its own agent session with its own context. This is recommended for private servers where it's just you and your bot.

**Add your server to the guild allowlist**

This enables your agent to respond in any channel on your server, not just DMs.

    **Ask your agent:**

> "Add my Discord Server ID `<server_id>` to the guild allowlist"
      
**Config:**

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      

**Allow responses without @mention**

By default, your agent only responds in guild channels when @mentioned. For a private server, you probably want it to respond to every message.

    In guild channels, normal assistant final replies stay private by default. Visible Discord output must be sent explicitly with the `message` tool, so the agent can lurk by default and only post when it decides a channel reply is useful.

    This means the selected model must reliably call tools. If Discord shows typing and the logs show token usage but no posted message, check the session log for assistant text with `didSendViaMessagingTool: false`. That means the model produced a private final answer instead of calling `message(action=send)`. Switch to a stronger tool-calling model, or use the config below to restore legacy automatic final replies.

    **Ask your agent:**

> "Allow my agent to respond on this server without having to be @mentioned"
      
**Config:**

Set `requireMention: false` in your guild config:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        To restore legacy automatic final replies for group/channel rooms, set `messages.groupChat.visibleReplies: "automatic"`.

      

**Plan for memory in guild channels**

By default, long-term memory (MEMORY.md) only loads in DM sessions. Guild channels do not auto-load MEMORY.md.

    **Ask your agent:**

> "When I ask questions in Discord channels, use memory_search or memory_get if you need long-term context from MEMORY.md."
      
**Manual:**

If you need shared context in every channel, put the stable instructions in `AGENTS.md` or `USER.md` (they are injected for every session). Keep long-term notes in `MEMORY.md` and access them on demand with memory tools.
      

Now create some channels on your Discord server and start chatting. Your agent can see the channel name, and each channel gets its own isolated session — so you can set up `#coding`, `#home`, `#research`, or whatever fits your workflow.

## Runtime model

- Gateway owns the Discord connection.
- Reply routing is deterministic: Discord inbound replies back to Discord.
- Discord guild/channel metadata is added to the model prompt as untrusted
  context, not as a user-visible reply prefix. If a model copies that envelope
  back, OpenClaw strips the copied metadata from outbound replies and from
  future replay context.
- By default (`session.dmScope=main`), direct chats share the agent main session (`agent:main:main`).
- Guild channels are isolated session keys (`agent:<agentId>:discord:channel:<channelId>`).
- Group DMs are ignored by default (`channels.discord.dm.groupEnabled=false`).
- Native slash commands run in isolated command sessions (`agent:<agentId>:discord:slash:<userId>`), while still carrying `CommandTargetSessionKey` to the routed conversation session.
- Text-only cron/heartbeat announce delivery to Discord uses the final
  assistant-visible answer once. Media and structured component payloads remain
  multi-message when the agent emits multiple deliverable payloads.

## Forum channels

Discord forum and media channels only accept thread posts. OpenClaw supports two ways to create them:

- Send a message to the forum parent (`channel:<forumId>`) to auto-create a thread. The thread title uses the first non-empty line of your message.
- Use `openclaw message thread create` to create a thread directly. Do not pass `--message-id` for forum channels.

Example: send to forum parent to create a thread

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

Example: create a forum thread explicitly

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

Forum parents do not accept Discord components. If you need components, send to the thread itself (`channel:<threadId>`).

## Interactive components

OpenClaw supports Discord components v2 containers for agent messages. Use the message tool with a `components` payload. Interaction results are routed back to the agent as normal inbound messages and follow the existing Discord `replyToMode` settings.

Supported blocks:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Action rows allow up to 5 buttons or a single select menu
- Select types: `string`, `user`, `role`, `mentionable`, `channel`

By default, components are single use. Set `components.reusable=true` to allow buttons, selects, and forms to be used multiple times until they expire.

To restrict who can click a button, set `allowedUsers` on that button (Discord user IDs, tags, or `*`). When configured, unmatched users receive an ephemeral denial.

The `/model` and `/models` slash commands open an interactive model picker with provider, model, and compatible runtime dropdowns plus a Submit step. `/models add` is deprecated and now returns a deprecation message instead of registering models from chat. The picker reply is ephemeral and only the invoking user can use it. Discord select menus are limited to 25 options, so add `provider/*` entries to `agents.defaults.models` when you want the picker to show dynamically discovered models only for selected providers such as `openai-codex` or `vllm`.

File attachments:

- `file` blocks must point to an attachment reference (`attachment://<filename>`)
- Provide the attachment via `media`/`path`/`filePath` (single file); use `media-gallery` for multiple files
- Use `filename` to override the upload name when 
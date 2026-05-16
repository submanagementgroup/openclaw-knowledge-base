---
domain: channels
topic: "Mattermost Channel Setup"
type: procedure
keywords:
  - Mattermost
  - mattermost bot
  - MATTERMOST_URL
  - mattermost channel
  - self-hosted chat
source: channels/mattermost.md
---

Status: downloadable plugin (bot token + WebSocket events). Channels, groups, and DMs are supported. Mattermost is a self-hostable team messaging platform; see the official site at [mattermost.com](https://mattermost.com) for product details and downloads.

## Install

Install Mattermost before configuring the channel:

**npm registry:**

```bash
    openclaw plugins install @openclaw/mattermost
    ```
  
**Local checkout:**

```bash
    openclaw plugins install ./path/to/local/mattermost-plugin
    ```
  
Details: [Plugins](/tools/plugin)

## Quick setup

**Ensure plugin is available**

Current packaged OpenClaw releases already bundle it. Older/custom installs can add it manually with the commands above.
  
**Create a Mattermost bot**

Create a Mattermost bot account and copy the **bot token**.
  
**Copy the base URL**

Copy the Mattermost **base URL** (e.g., `https://chat.example.com`).
  
**Configure OpenClaw and start the gateway**

Minimal config:

    ```json5
    {
      channels: {
        mattermost: {
          enabled: true,
          botToken: "mm-token",
          baseUrl: "https://chat.example.com",
          dmPolicy: "pairing",
        },
      },
    }
    ```

  
## Native slash commands

Native slash commands are opt-in. When enabled, OpenClaw registers `oc_*` slash commands via the Mattermost API and receives callback POSTs on the gateway HTTP server.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Use when Mattermost cannot reach the gateway directly (reverse proxy/public URL).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

### Behavior notes

- `native: "auto"` defaults to disabled for Mattermost. Set `native: true` to enable.
    - If `callbackUrl` is omitted, OpenClaw derives one from gateway host/port + `callbackPath`.
    - For multi-account setups, `commands` can be set at the top level or under `channels.mattermost.accounts.<id>.commands` (account values override top-level fields).
    - Command callbacks are validated with the per-command tokens returned by Mattermost when OpenClaw registers `oc_*` commands.
    - OpenClaw refreshes current Mattermost command registration before accepting each callback so stale tokens from deleted or regenerated slash commands stop being accepted without a gateway restart.
    - Callback validation fails closed if the Mattermost API cannot confirm the command is still current; failed validations are cached briefly, concurrent lookups are coalesced, and fresh lookup starts are rate-limited per command to bound replay pressure.
    - Slash callbacks fail closed when registration failed, startup was partial, or the callback token does not match the resolved command's registered token (a token valid for one command cannot reach upstream validation for a different command).

  ### Reachability requirement

The callback endpoint must be reachable from the Mattermost server.

    - Do not set `callbackUrl` to `localhost` unless Mattermost runs on the same host/network namespace as OpenClaw.
    - Do not set `callbackUrl` to your Mattermost base URL unless that URL reverse-proxies `/api/channels/mattermost/command` to OpenClaw.
    - A quick check is `curl https://<gateway-host>/api/channels/mattermost/command`; a GET should return `405 Method Not Allowed` from OpenClaw, not `404`.

  ### Mattermost egress allowlist

If your callback targets private/tailnet/internal addresses, set Mattermost `ServiceSettings.AllowedUntrustedInternalConnections` to include the callback host/domain.

    Use host/domain entries, not full URLs.

    - Good: `gateway.tailnet-name.ts.net`
    - Bad: `https://gateway.tailnet-name.ts.net`

  ## Environment variables (default account)

Set these on the gateway host if you prefer env vars:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

> **Note:** Env vars apply only to the **default** account (`default`). Other accounts must use config values.

`MATTERMOST_URL` cannot be set from a workspace `.env`; see [Workspace `.env` files](/gateway/security).


## Chat modes

Mattermost responds to DMs automatically. Channel behavior is controlled by `chatmode`:

**oncall (default):**

Respond only when @mentioned in channels.
  
**onmessage:**

Respond to every channel message.
  
**onchar:**

Respond when a message starts with a trigger prefix.
  
Config example:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"],
    },
  },
}
```

Notes:

- `onchar` still responds to explicit @mentions.
- `channels.mattermost.requireMention` is honored for legacy configs but `chatmode` is preferred.

## Threading and sessions

Use `channels.mattermost.replyToMode` to control whether channel and group replies stay in the main channel or start a thread under the triggering post.

- `off` (default): only reply in a thread when the inbound post is already in one.
- `first`: for top-level channel/group posts, start a thread under that post and route the conversation to a thread-scoped session.
- `all`: same behavior as `first` for Mattermost today.
- Direct messages ignore this setting and stay non-threaded.

Config example:

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
    },
  },
}
```

Notes:

- Thread-scoped sessions use the triggering post id as the thread root.
- `first` and `all` are currently equivalent because once Mattermost has a thread root, follow-up chunks and media continue in that same thread.

## Access control (DMs)

- Default: `channels.mattermost.dmPolicy = "pairing"` (unknown senders get a pairing code).
- Approve via:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost `
- Public DMs: `channels.mattermost.dmPolicy="open"` plus `channels.mattermost.allowFrom=["*"]`.
- `channels.mattermost.allowFrom` accepts `accessGroup:<name>` entries. See [Access groups](/channels/access-groups).

## Channels (groups)

- Default: `channels.mattermost.groupPolicy = "allowlist"` (mention-gated).
- Allowlist senders with `channels.mattermost.groupAllowFrom` (user IDs recommended).
- `channels.mattermost.groupAllowFrom` accepts `accessGroup:<name>` entries. See [Access groups](/channels/access-groups).
- Per-channel mention overrides live under `channels.mattermost.groups.<channelId>.requireMention` or `channels.mattermost.groups["*"].requireMention` for a default.
- `@username` matching is mutable and only enabled when `channels.mattermost.dangerouslyAllowNameMatching: true`.
- Open channels: `channels.mattermost.groupPolicy="open"` (mention-gated).
- Runtime note: if `channels.mattermost` is completely missing, runtime falls back to `groupPolicy="allowlist"` for group checks (even if `channels.defaults.groupPolicy` is set).

Example:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## Targets for outbound delivery

Use these target formats with `openclaw message send` or cron/webhooks:

- `channel:<id>` for a channel
- `user:<id>` for a DM
- `@username` for a DM (resolved via the Mattermost API)

> **Note:** Bare opaque IDs (like `64ifufp...`) are **ambiguous** in Mattermost (user ID vs channel ID).

OpenClaw resolves them **user-first**:

- If the ID exists as a user (`GET /api/v4/users/<id>` succeeds), OpenClaw sends a **DM** by resolving the direct channel via `/api/v4/channels/direct`.
- Otherwise the ID is treated as a **channel ID**.

If you need deterministic behavior, always use the explicit prefixes (`user:<id>` / `channel:<id>`).


## DM channel retry

When OpenClaw sends to a Mattermost DM target and needs to resolve the direct channel first, it retries transient direct-channel creation failures by default.

Use `channels.mattermost.dmChannelRetry` to tune that behavior globally for the Mattermost plugin, or `channels.mattermost.accounts.<id>.dmChannelRetry` for one account.

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

Notes:

- This applies only to DM channel creation (`/api/v4/channels/direct`), not every Mattermost API call.
- Retries apply to transient failures such as rate limits, 5xx responses, and network or timeout errors.
- 4xx client errors other than `429` are treated as permanent and are not retried.

## Preview streaming

Mattermost streams thinking, tool activity, and partial reply text into a single **draft preview post** that finalizes in place when the final answer is safe to send. The preview updates on the same post id instead of spamming the channel with per-chunk messages. Media/error finals cancel pending preview edits and use normal delivery instead of flushing a throwaway preview post.

Enable via `channels.mattermost.streaming`:

```json5
{
  channels: {
    mattermost: {
      streaming: "partial", // off | partial | block | progress
    },
  },
}
```

### Streaming modes

- `partial` is the usual choice: one preview post that is edited as the reply grows, then finalized with the complete answer.
    - `block` uses append-style draft chunks inside the preview post.
    - `progress` shows a status preview while generating and only posts the final answer at completion.
    - `off` disables preview streaming.

  ### Streaming behavior notes

- If the stream cannot be finalized in place (for example the post was deleted mid-stream), OpenClaw falls back to sending a fresh final post so the reply is never lost.
    - Reasoning-only payloads are suppressed from channel posts, including text that arrives as a `> Reasoning:` blockquote. Set `/reasoning on` to see thinking in other surfaces; the Mattermost final post keeps the answer only.
    - See [Streaming](/concepts/streaming#preview-streaming-modes) for the channel-mapping matrix.

  ## Reactions (message tool)

- Use `message action=react` with `channel=mattermost`.
- `messageId` is the Mattermost post id.
- `emoji` accepts names like `thumbsup` or `:+1:` (colons are optional).
- Set `remove=true` (boolean) to remove a reaction.
- Reaction add/remove events are forwarded as system events to the routed agent session.

Examples:

```
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

Config:

- `channels.mattermost.actions.reactions`: enable/disable reaction actions (default true).
- Per-account override: `channels.mattermost.accounts.<id>.actions.reactions`.

## Interactive buttons (message tool)

Send messages with clickable buttons. When a user clicks a button, the agent receives the selection and can respond.

Enable buttons by adding `inlineButtons` to the channel capabilities:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

Use `message action=send` with a `buttons` parameter. Buttons are a 2D array (rows of buttons):

```
message action=send channel=mattermost target=channel:<channelId> buttons=[[{"text":"Yes","callback_data":"yes"},{"text":"No","callback_data":"no"}]]
```

Button fields:


  Display label.


  Value sent back on click (used as the action ID).


  Button style.


When a user clicks a button:

**Buttons replaced with confirmation**

All buttons are replaced with a confirmation line (e.g., "✓ **Yes** selected by @user").
  
**Agent receives the selection**

The agent receives the selection as an inbound message and responds.
  
### Implementation notes

- Button callbacks use HMAC-SHA256 verification (automatic, no config needed).
    - Mattermost strips callb
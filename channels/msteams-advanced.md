---
domain: channels
topic: "Microsoft Teams Advanced Configuration"
type: reference
keywords:
  - Microsoft Teams
  - Teams advanced
  - Teams adaptive cards
  - Teams permissions
source: channels/msteams.md
---

ecret over network      | Certificate management overhead       |
| **Managed Identity** | `authType: "federated"` + `useManagedIdentity` | Passwordless, no secrets to manage | Azure infrastructure required         |

**Default behavior:** When `authType` is not set, OpenClaw defaults to client secret authentication. Existing configurations continue to work without changes.

## Local development (tunneling)

Teams can't reach `localhost`. Use a persistent dev tunnel so your URL stays the same across sessions:

```bash
# One-time setup:
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Each dev session:
devtunnel host my-openclaw-bot
```

Alternatives: `ngrok http 3978` or `tailscale funnel 3978` (URLs may change each session).

If your tunnel URL changes, update the endpoint:

```bash
teams app update <teamsAppId> --endpoint "https://<new-url>/api/messages"
```

## Testing the Bot

**Run diagnostics:**

```bash
teams app doctor <teamsAppId>
```

Checks bot registration, AAD app, manifest, and SSO configuration in one pass.

**Send a test message:**

1. Install the Teams app (use the install link from `teams app get <id> --install-link`)
2. Find the bot in Teams and send a DM
3. Check gateway logs for incoming activity

## Environment variables

All config keys can be set via environment variables instead:

- `MSTEAMS_APP_ID`
- `MSTEAMS_APP_PASSWORD`
- `MSTEAMS_TENANT_ID`
- `MSTEAMS_AUTH_TYPE` (optional: `"secret"` or `"federated"`)
- `MSTEAMS_CERTIFICATE_PATH` (federated + certificate)
- `MSTEAMS_CERTIFICATE_THUMBPRINT` (optional, not required for auth)
- `MSTEAMS_USE_MANAGED_IDENTITY` (federated + managed identity)
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID` (user-assigned MI only)

## Member info action

OpenClaw exposes a Graph-backed `member-info` action for Microsoft Teams so agents and automations can resolve channel member details (display name, email, role) directly from Microsoft Graph.

Requirements:

- `Member.Read.Group` RSC permission (already in the recommended manifest)
- For cross-team lookups: `User.Read.All` Graph Application permission with admin consent

The action is gated by `channels.msteams.actions.memberInfo` (default: enabled when Graph credentials are available).

## History context

- `channels.msteams.historyLimit` controls how many recent channel/group messages are wrapped into the prompt.
- Falls back to `messages.groupChat.historyLimit`. Set `0` to disable (default 50).
- Fetched thread history is filtered by sender allowlists (`allowFrom` / `groupAllowFrom`), so thread context seeding only includes messages from allowed senders.
- Quoted attachment context (`ReplyTo*` derived from Teams reply HTML) is currently passed as received.
- In other words, allowlists gate who can trigger the agent; only specific supplemental context paths are filtered today.
- DM history can be limited with `channels.msteams.dmHistoryLimit` (user turns). Per-user overrides: `channels.msteams.dms["<user_id>"].historyLimit`.

## Current Teams RSC permissions (manifest)

These are the **existing resourceSpecific permissions** in our Teams app manifest. They only apply inside the team/chat where the app is installed.

**For channels (team scope):**

- `ChannelMessage.Read.Group` (Application) - receive all channel messages without @mention
- `ChannelMessage.Send.Group` (Application)
- `Member.Read.Group` (Application)
- `Owner.Read.Group` (Application)
- `ChannelSettings.Read.Group` (Application)
- `TeamMember.Read.Group` (Application)
- `TeamSettings.Read.Group` (Application)

**For group chats:**

- `ChatMessage.Read.Chat` (Application) - receive all group chat messages without @mention

To add RSC permissions via the Teams CLI:

```bash
teams app rsc add <teamsAppId> ChannelMessage.Read.Group --type Application
```

## Example Teams manifest (redacted)

Minimal, valid example with the required fields. Replace IDs and URLs.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Your Org",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw in Teams", full: "OpenClaw in Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### Manifest caveats (must-have fields)

- `bots[].botId` **must** match the Azure Bot App ID.
- `webApplicationInfo.id` **must** match the Azure Bot App ID.
- `bots[].scopes` must include the surfaces you plan to use (`personal`, `team`, `groupChat`).
- `bots[].supportsFiles: true` is required for file handling in personal scope.
- `authorization.permissions.resourceSpecific` must include channel read/send if you want channel traffic.

### Updating an existing app

To update an already-installed Teams app (e.g., to add RSC permissions):

```bash
# Download, edit, and re-upload the manifest
teams app manifest download <teamsAppId> manifest.json
# Edit manifest.json locally...
teams app manifest upload manifest.json <teamsAppId>
# Version is auto-bumped if content changed
```

After updating, reinstall the app in each team for new permissions to take effect, and **fully quit and relaunch Teams** (not just close the window) to clear cached app metadata.

<details>
<summary>Manual manifest update (without CLI)</summary>

1. Update your `manifest.json` with the new settings
2. **Increment the `version` field** (e.g., `1.0.0` → `1.1.0`)
3. **Re-zip** the manifest with icons (`manifest.json`, `outline.png`, `color.png`)
4. Upload the new zip:
   - **Teams Admin Center:** Teams apps → Manage apps → find your app → Upload new version
   - **Sideload:** In Teams → Apps → Manage your apps → Upload a custom app

</details>

## Capabilities: RSC only vs Graph

### With **Teams RSC only** (app installed, no Graph API permissions)

Works:

- Read channel message **text** content.
- Send channel message **text** content.
- Receive **personal (DM)** file attachments.

Does NOT work:

- Channel/group **image or file contents** (payload only includes HTML stub).
- Downloading attachments stored in SharePoint/OneDrive.
- Reading message history (beyond the live webhook event).

### With **Teams RSC + Microsoft Graph Application permissions**

Adds:

- Downloading hosted contents (images pasted into messages).
- Downloading file attachments stored in SharePoint/OneDrive.
- Reading channel/chat message history via Graph.

### RSC vs Graph API

| Capability              | RSC Permissions      | Graph API                           |
| ----------------------- | -------------------- | ----------------------------------- |
| **Real-time messages**  | Yes (via webhook)    | No (polling only)                   |
| **Historical messages** | No                   | Yes (can query history)             |
| **Setup complexity**    | App manifest only    | Requires admin consent + token flow |
| **Works offline**       | No (must be running) | Yes (query anytime)                 |

**Bottom line:** RSC is for real-time listening; Graph API is for historical access. For catching up on missed messages while offline, you need Graph API with `ChannelMessage.Read.All` (requires admin consent).

## Graph-enabled media + history (required for channels)

If you need images/files in **channels** or want to fetch **message history**, you must enable Microsoft Graph permissions and grant admin consent.

1. In Entra ID (Azure AD) **App Registration**, add Microsoft Graph **Application permissions**:
   - `ChannelMessage.Read.All` (channel attachments + history)
   - `Chat.Read.All` or `ChatMessage.Read.All` (group chats)
2. **Grant admin consent** for the tenant.
3. Bump the Teams app **manifest version**, re-upload, and **reinstall the app in Teams**.
4. **Fully quit and relaunch Teams** to clear cached app metadata.

**Additional permission for user mentions:** User @mentions work out of the box for users in the conversation. However, if you want to dynamically search and mention users who are **not in the current conversation**, add `User.Read.All` (Application) permission and grant admin consent.

## Known limitations

### Webhook timeouts

Teams delivers messages via HTTP webhook. If processing takes too long (e.g., slow LLM responses), you may see:

- Gateway timeouts
- Teams retrying the message (causing duplicates)
- Dropped replies

OpenClaw handles this by returning quickly and sending replies proactively, but very slow responses may still cause issues.

### Formatting

Teams markdown is more limited than Slack or Discord:

- Basic formatting works: **bold**, _italic_, `code`, links
- Complex markdown (tables, nested lists) may not render correctly
- Adaptive Cards are supported for polls and semantic presentation sends (see below)

## Configuration

Key settings (see `/gateway/configuration` for shared channel patterns):

- `channels.msteams.enabled`: enable/disable the channel.
- `channels.msteams.appId`, `channels.msteams.appPassword`, `channels.msteams.tenantId`: bot credentials.
- `channels.msteams.webhook.port` (default `3978`)
- `channels.msteams.webhook.path` (default `/api/messages`)
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled` (default: pairing)
- `channels.msteams.allowFrom`: DM allowlist (AAD object IDs recommended). The wizard resolves names to IDs during setup when Graph access is available.
- `channels.msteams.dangerouslyAllowNameMatching`: break-glass toggle to re-enable mutable UPN/display-name matching and direct team/channel name routing.
- `channels.msteams.textChunkLimit`: outbound text chunk size.
- `channels.msteams.chunkMode`: `length` (default) or `newline` to split on blank lines (paragraph boundaries) before length chunking.
- `channels.msteams.mediaAllowHosts`: allowlist for inbound attachment hosts (defaults to Microsoft/Teams domains).
- `channels.msteams.mediaAuthAllowHosts`: allowlist for attaching Authorization headers on media retries (defaults to Graph + Bot Framework hosts).
- `channels.msteams.requireMention`: require @mention in channels/groups (default true).
- `channels.msteams.replyStyle`: `thread | top-level` (see [Reply Style](#reply-style-threads-vs-posts)).
- `channels.msteams.teams.<teamId>.replyStyle`: per-team override.
- `channels.msteams.teams.<teamId>.requireMention`: per-team override.
- `channels.msteams.teams.<teamId>.tools`: default per-team tool policy overrides (`allow`/`deny`/`alsoAllow`) used when a channel override is missing.
- `channels.msteams.teams.<teamId>.toolsBySender`: default per-team per-sender tool policy overrides (`"*"` wildcard supported).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: per-channel override.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: per-channel override.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: per-channel tool policy overrides (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: per-channel per-sender tool policy overrides (`"*"` wildcard supported).
- `toolsBySender` keys should use explicit prefixes:
  `channel:`, `id:`, `e164:`, `username:`, `name:` (legacy unprefixed keys still map to `id:` only).
- `channels.msteams.actions.memberInfo`: enable or disable the Graph-backed member info action (default: enabled when Graph credentials are available).
- `channels.msteams.authType`: authentication type - `"secret"` (default) or `"federated"`.
- `channels.msteams.certificatePath`: path to PEM certificate file (federated + certificate auth).
- `channels.msteams.certificateThumbprint`: certificate thumbprint (optional, not required for auth).
- `channels.msteams.useManagedIdentity`: enable managed identity auth (federated mode).
- `channels.msteams.managedIdentityClientId`: client ID for user-assigned managed identity.
- `channels.msteams.sharePointSiteId`: SharePoint site ID for file uploads in group chats/channels (see [Sending files in group chats](#sending-files-in-group-chats)).

## Routing and sessions

- Session keys follow the standard agent format (see [/concepts/session](/concepts/session)):
  - Direct messages share the main session (`agent:<agentId>:<mainKey>`).
  - Channel/group messages use conversation id:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## Reply style: threads vs posts

Teams recently introduced two channel UI styles over the same underlying data model:

| Style                    | Description                                               | Recommended `replyStyle` |
| ------------------------ | --------------------------------------------------------- | ------------------------ |
| **Posts** (classic)      | Messages appear as cards with threaded replies
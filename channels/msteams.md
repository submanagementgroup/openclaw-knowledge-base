---
domain: channels
topic: "Microsoft Teams Channel Setup"
type: procedure
keywords:
  - Microsoft Teams
  - MS Teams
  - Teams bot
  - MSTEAMS_APP_ID
  - Teams channel
  - teams setup
source: channels/msteams.md
---

Status: text + DM attachments are supported; channel/group file sending requires `sharePointSiteId` + Graph permissions (see [Sending files in group chats](#sending-files-in-group-chats)). Polls are sent via Adaptive Cards. Message actions expose explicit `upload-file` for file-first sends.

## Bundled plugin

Microsoft Teams ships as a bundled plugin in current OpenClaw releases, so no
separate install is required in the normal packaged build.

If you are on an older build or a custom install that excludes bundled Teams,
install the npm package directly:

```bash
openclaw plugins install @openclaw/msteams
```

Use the bare package to follow the current official release tag. Pin an exact
version only when you need a reproducible install.

Local checkout (when running from a git repo):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

Details: [Plugins](/tools/plugin)

## Quick setup

The [`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli) handles bot registration, manifest creation, and credential generation in a single command.

**1. Install and log in**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # verify you're logged in and see your tenant info
```

> **Note:** The Teams CLI is currently in preview. Commands and flags may change between releases.


**2. Start a tunnel** (Teams can't reach localhost)

Install and authenticate the devtunnel CLI if you haven't already ([getting started guide](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)).

```bash
# One-time setup (persistent URL across sessions):
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Each dev session:
devtunnel host my-openclaw-bot
# Your endpoint: https://<tunnel-id>.devtunnels.ms/api/messages
```

> **Note:** `--allow-anonymous` is required because Teams cannot authenticate with devtunnels. Each incoming bot request is still validated by the Teams SDK automatically.


Alternatives: `ngrok http 3978` or `tailscale funnel 3978` (but these may change URLs each session).

**3. Create the app**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

This single command:

- Creates an Entra ID (Azure AD) application
- Generates a client secret
- Builds and uploads a Teams app manifest (with icons)
- Registers the bot (Teams-managed by default - no Azure subscription needed)

The output will show `CLIENT_ID`, `CLIENT_SECRET`, `TENANT_ID`, and a **Teams App ID** - note these for the next steps. It also offers to install the app in Teams directly.

**4. Configure OpenClaw** using the credentials from the output:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "",
      appPassword: "",
      tenantId: "",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Or use environment variables directly: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

**5. Install the app in Teams**

`teams app create` will prompt you to install the app - select "Install in Teams". If you skipped it, you can get the link later:

```bash
teams app get <teamsAppId> --install-link
```

**6. Verify everything works**

```bash
teams app doctor <teamsAppId>
```

This runs diagnostics across bot registration, AAD app config, manifest validity, and SSO setup.

For production deployments, consider using [federated authentication](/channels/msteams#federated-authentication-certificate-plus-managed-identity) (certificate or managed identity) instead of client secrets.

> **Note:** Group chats are blocked by default (`channels.msteams.groupPolicy: "allowlist"`). To allow group replies, set `channels.msteams.groupAllowFrom`, or use `groupPolicy: "open"` to allow any member (mention-gated).


## Goals

- Talk to OpenClaw via Teams DMs, group chats, or channels.
- Keep routing deterministic: replies always go back to the channel they arrived on.
- Default to safe channel behavior (mentions required unless configured otherwise).

## Config writes

By default, Microsoft Teams is allowed to write config updates triggered by `/config set|unset` (requires `commands.config: true`).

Disable with:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## Access control (DMs + groups)

**DM access**

- Default: `channels.msteams.dmPolicy = "pairing"`. Unknown senders are ignored until approved.
- `channels.msteams.allowFrom` should use stable AAD object IDs or static sender access groups such as `accessGroup:core-team`.
- Do not rely on UPN/display-name matching for allowlists - they can change. OpenClaw disables direct name matching by default; opt in explicitly with `channels.msteams.dangerouslyAllowNameMatching: true`.
- The wizard can resolve names to IDs via Microsoft Graph when credentials allow.

**Group access**

- Default: `channels.msteams.groupPolicy = "allowlist"` (blocked unless you add `groupAllowFrom`). Use `channels.defaults.groupPolicy` to override the default when unset.
- `channels.msteams.groupAllowFrom` controls which senders or static sender access groups can trigger in group chats/channels (falls back to `channels.msteams.allowFrom`).
- Set `groupPolicy: "open"` to allow any member (still mention-gated by default).
- To allow **no channels**, set `channels.msteams.groupPolicy: "disabled"`.

Example:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["00000000-0000-0000-0000-000000000000", "accessGroup:core-team"],
    },
  },
}
```

**Teams + channel allowlist**

- Scope group/channel replies by listing teams and channels under `channels.msteams.teams`.
- Keys should use stable Teams conversation IDs from Teams links, not mutable display names.
- When `groupPolicy="allowlist"` and a teams allowlist is present, only listed teams/channels are accepted (mention-gated).
- The configure wizard accepts `Team/Channel` entries and stores them for you.
- On startup, OpenClaw resolves team/channel and user allowlist names to IDs (when Graph permissions allow)
  and logs the mapping; unresolved team/channel names are kept as typed but ignored for routing by default unless `channels.msteams.dangerouslyAllowNameMatching: true` is enabled.

Example:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>Manual setup (without the Teams CLI)</strong></summary>

If you can't use the Teams CLI, you can set up the bot manually through the Azure Portal.

### How it works

1. Ensure the Microsoft Teams plugin is available (bundled in current releases).
2. Create an **Azure Bot** (App ID + secret + tenant ID).
3. Build a **Teams app package** that references the bot and includes the RSC permissions below.
4. Upload/install the Teams app into a team (or personal scope for DMs).
5. Configure `msteams` in `~/.openclaw/openclaw.json` (or env vars) and start the gateway.
6. The gateway listens for Bot Framework webhook traffic on `/api/messages` by default.

### Step 1: Create Azure Bot

1. Go to [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot)
2. Fill in the **Basics** tab:

   | Field              | Value                                                    |
   | ------------------ | -------------------------------------------------------- |
   | **Bot handle**     | Your bot name, e.g., `openclaw-msteams` (must be unique) |
   | **Subscription**   | Select your Azure subscription                           |
   | **Resource group** | Create new or use existing                               |
   | **Pricing tier**   | **Free** for dev/testing                                 |
   | **Type of App**    | **Single Tenant** (recommended - see note below)         |
   | **Creation type**  | **Create new Microsoft App ID**                          |

> **Note:** Creation of new multi-tenant bots was deprecated after 2025-07-31. Use **Single Tenant** for new bots.


3. Click **Review + create** → **Create** (wait ~1-2 minutes)

### Step 2: Get Credentials

1. Go to your Azure Bot resource → **Configuration**
2. Copy **Microsoft App ID** → this is your `appId`
3. Click **Manage Password** → go to the App Registration
4. Under **Certificates & secrets** → **New client secret** → copy the **Value** → this is your `appPassword`
5. Go to **Overview** → copy **Directory (tenant) ID** → this is your `tenantId`

### Step 3: Configure Messaging Endpoint

1. In Azure Bot → **Configuration**
2. Set **Messaging endpoint** to your webhook URL:
   - Production: `https://your-domain.com/api/messages`
   - Local dev: Use a tunnel (see [Local Development](#local-development-tunneling) below)

### Step 4: Enable Teams Channel

1. In Azure Bot → **Channels**
2. Click **Microsoft Teams** → Configure → Save
3. Accept the Terms of Service

### Step 5: Build Teams App Manifest

- Include a `bot` entry with `botId = `.
- Scopes: `personal`, `team`, `groupChat`.
- `supportsFiles: true` (required for personal scope file handling).
- Add RSC permissions (see [RSC Permissions](#current-teams-rsc-permissions-manifest)).
- Create icons: `outline.png` (32x32) and `color.png` (192x192).
- Zip all three files together: `manifest.json`, `outline.png`, `color.png`.

### Step 6: Configure OpenClaw

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "",
      appPassword: "",
      tenantId: "",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Environment variables: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

### Step 7: Run the Gateway

The Teams channel starts automatically when the plugin is available and `msteams` config exists with credentials.

</details>

## Federated authentication (certificate plus managed identity)

> Added in 2026.4.11

For production deployments, OpenClaw supports **federated authentication** as a more secure alternative to client secrets. Two methods are available:

### Option A: Certificate-based authentication

Use a PEM certificate registered with your Entra ID app registration.

**Setup:**

1. Generate or obtain a certificate (PEM format with private key).
2. In Entra ID → App Registration → **Certificates & secrets** → **Certificates** → Upload the public certificate.

**Config:**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "",
      tenantId: "",
      authType: "federated",
      certificatePath: "/path/to/cert.pem",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Env vars:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_CERTIFICATE_PATH=/path/to/cert.pem`

### Option B: Azure Managed Identity

Use Azure Managed Identity for passwordless authentication. This is ideal for deployments on Azure infrastructure (AKS, App Service, Azure VMs) where a managed identity is available.

**How it works:**

1. The bot pod/VM has a managed identity (system-assigned or user-assigned).
2. A **federated identity credential** links the managed identity to the Entra ID app registration.
3. At runtime, OpenClaw uses `@azure/identity` to acquire tokens from the Azure IMDS endpoint (`169.254.169.254`).
4. The token is passed to the Teams SDK for bot authentication.

**Prerequisites:**

- Azure infrastructure with managed identity enabled (AKS workload identity, App Service, VM)
- Federated identity credential created on the Entra ID app registration
- Network access to IMDS (`169.254.169.254:80`) from the pod/VM

**Config (system-assigned managed identity):**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "",
      tenantId: "",
      authType: "federated",
      useManagedIdentity: true,
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Config (user-assigned managed identity):**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "",
      tenantId: "",
      authType: "federated",
      useManagedIdentity: true,
      managedIdentityClientId: "",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Env vars:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_USE_MANAGED_IDENTITY=true`
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID=<client-id>` (only for user-assigned)

### AKS Workload Identity Setup

For AKS deployments using workload identity:

1. **Enable workload identity** on your AKS cluster.
2. **Create a federated identity credential** on the Entra ID app registration:

   ```bash
   az ad app federated-credential create --id  --parameters '{
     "name": "my-bot-workload-identity",
     "issuer": "",
     "subject": "system:serviceaccount::",
     "audiences": ["api://AzureADTokenExchange"]
   }'
   ```

3. **Annotate the Kubernetes service account** with the app client ID:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-bot-sa
     annotations:
       azure.workload.identity/client-id: ""
   ```

4. **Label the pod** for workload identity injection:

   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

5. **Ensure network access** to IMDS (`169.254.169.254`) - if using NetworkPolicy, add an egress rule allowing traffic to `169.254.169.254/32` on port 80.

### Auth type comparison

| Method               | Config                                         | Pros                               | Cons                                  |
| -------------------- | ---------------------------------------------- | ---------------------------------- | ------------------------------------- |
| **Client secret**    | `appPassword`                                  | Simple setup                       | Secret rotation required, less secure |
| **Certificate**      | `authType: "federated"` + `certificatePath`    | No shared s
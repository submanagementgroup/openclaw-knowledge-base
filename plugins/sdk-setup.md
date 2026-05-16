---
domain: plugins
topic: "Plugin SDK Setup and Getting Started"
type: procedure
keywords:
  - SDK setup
  - plugin setup
  - npm
  - plugin scaffold
  - create plugin
  - sdk-setup
source: plugins/sdk-setup.md
---

Reference for plugin packaging (`package.json` metadata), manifests (`openclaw.plugin.json`), setup entries, and config schemas.

> **Note:** **Looking for a walkthrough?** The how-to guides cover packaging in context: [Channel plugins](/plugins/sdk-channel-plugins#step-1-package-and-manifest) and [Provider plugins](/plugins/sdk-provider-plugins#step-1-package-and-manifest).


## Package metadata

Your `package.json` needs an `openclaw` field that tells the plugin system what your plugin provides:

**Channel plugin:**

```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "My Channel",
          "blurb": "Short description of the channel."
        }
      }
    }
    ```
  
**Provider plugin / ClawHub baseline:**

```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```
  
> **Note:** If you publish the plugin externally on ClawHub, those `compat` and `build` fields are required. The canonical publish snippets live in `docs/snippets/plugin-publish/`.


### `openclaw` fields


  Entry point files (relative to package root).


  Lightweight setup-only entry (optional).


  Channel catalog metadata for setup, picker, quickstart, and status surfaces.


  Provider ids registered by this plugin.


  Install hints: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`.


  Startup behavior flags.


### `openclaw.channel`

`openclaw.channel` is cheap package metadata for channel discovery and setup surfaces before runtime loads.

| Field                                  | Type       | What it means                                                                 |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | Canonical channel id.                                                         |
| `label`                                | `string`   | Primary channel label.                                                        |
| `selectionLabel`                       | `string`   | Picker/setup label when it should differ from `label`.                        |
| `detailLabel`                          | `string`   | Secondary detail label for richer channel catalogs and status surfaces.       |
| `docsPath`                             | `string`   | Docs path for setup and selection links.                                      |
| `docsLabel`                            | `string`   | Override label used for docs links when it should differ from the channel id. |
| `blurb`                                | `string`   | Short onboarding/catalog description.                                         |
| `order`                                | `number`   | Sort order in channel catalogs.                                               |
| `aliases`                              | `string[]` | Extra lookup aliases for channel selection.                                   |
| `preferOver`                           | `string[]` | Lower-priority plugin/channel ids this channel should outrank.                |
| `systemImage`                          | `string`   | Optional icon/system-image name for channel UI catalogs.                      |
| `selectionDocsPrefix`                  | `string`   | Prefix text before docs links in selection surfaces.                          |
| `selectionDocsOmitLabel`               | `boolean`  | Show the docs path directly instead of a labeled docs link in selection copy. |
| `selectionExtras`                      | `string[]` | Extra short strings appended in selection copy.                               |
| `markdownCapable`                      | `boolean`  | Marks the channel as markdown-capable for outbound formatting decisions.      |
| `exposure`                             | `object`   | Channel visibility controls for setup, configured lists, and docs surfaces.   |
| `quickstartAllowFrom`                  | `boolean`  | Opt this channel into the standard quickstart `allowFrom` setup flow.         |
| `forceAccountBinding`                  | `boolean`  | Require explicit account binding even when only one account exists.           |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Prefer session lookup when resolving announce targets for this channel.       |

Example:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` supports:

- `configured`: include the channel in configured/status-style listing surfaces
- `setup`: include the channel in interactive setup/configure pickers
- `docs`: mark the channel as public-facing in docs/navigation surfaces

> **Note:** `showConfigured` and `showInSetup` remain supported as legacy aliases. Prefer `exposure`.


### `openclaw.install`

`openclaw.install` is package metadata, not manifest metadata.

| Field                        | Type                                | What it means                                                                     |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | Canonical ClawHub spec for install/update and onboarding install-on-demand flows. |
| `npmSpec`                    | `string`                            | Canonical npm spec for install/update fallback flows.                             |
| `localPath`                  | `string`                            | Local development or bundled install path.                                        |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | Preferred install source when multiple sources are available.                     |
| `minHostVersion`             | `string`                            | Minimum supported OpenClaw version in the form `>=x.y.z` or `>=x.y.z-prerelease`. |
| `expectedIntegrity`          | `string`                            | Expected npm dist integrity string, usually `sha512-...`, for pinned installs.    |
| `allowInvalidConfigRecovery` | `boolean`                           | Lets bundled-plugin reinstall flows recover from specific stale-config failures.  |

### Onboarding behavior

Interactive onboarding also uses `openclaw.install` for install-on-demand surfaces. If your plugin exposes provider auth choices or channel setup/catalog metadata before runtime loads, onboarding can show that choice, prompt for ClawHub, npm, or local install, install or enable the plugin, then continue the selected flow. ClawHub onboarding choices use `clawhubSpec` and are preferred when present; npm choices require trusted catalog metadata with a registry `npmSpec`; exact versions and `expectedIntegrity` are optional npm pins. If `expectedIntegrity` is present, install/update flows enforce it for npm. Keep the "what to show" metadata in `openclaw.plugin.json` and the "how to install it" metadata in `package.json`.
  ### minHostVersion enforcement

If `minHostVersion` is set, install and non-bundled manifest-registry loading both enforce it. Older hosts skip external plugins; invalid version strings are rejected. Bundled source plugins are assumed to be co-versioned with the host checkout.
  ### Pinned npm installs

For pinned npm installs, keep the exact version in `npmSpec` and add the expected artifact integrity:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  ### allowInvalidConfigRecovery scope

`allowInvalidConfigRecovery` is not a general bypass for broken configs. It is for narrow bundled-plugin recovery only, so reinstall/setup can repair known upgrade leftovers like a missing bundled plugin path or stale `channels.<id>` entry for that same plugin. If config is broken for unrelated reasons, install still fails closed and tells the operator to run `openclaw doctor --fix`.
  ### Deferred full load

Channel plugins can opt into deferred loading with:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

When enabled, OpenClaw loads only `setupEntry` during the pre-listen startup phase, even for already-configured channels. The full entry loads after the gateway starts listening.

> **Note:** Only enable deferred loading when your `setupEntry` registers everything the gateway needs before it starts listening (channel registration, HTTP routes, gateway methods). If the full entry owns required startup capabilities, keep the default behavior.


If your setup/full entry registers gateway RPC methods, keep them on a plugin-specific prefix. Reserved core admin namespaces (`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`) stay core-owned and always resolve to `operator.admin`.

## Plugin manifest

Every native plugin must ship an `openclaw.plugin.json` in the package root. OpenClaw uses this to validate config without executing plugin code.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

For channel plugins, add `kind` and `channels`:

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Even plugins with no config must ship a schema. An empty schema is valid:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

See [Plugin manifest](/plugins/manifest) for the full schema reference.

## ClawHub publishing

For plugin packages, use the package-specific ClawHub command:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

> **Note:** The legacy skill-only publish alias is for skills. Plugin packages should always use `clawhub package publish`.


## Setup entry

The `setup-entry.ts` file is a lightweight alternative to `index.ts` that OpenClaw loads when it only needs setup surfaces (onboarding, config repair, disabled channel inspection).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/
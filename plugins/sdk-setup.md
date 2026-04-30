---
domain: plugins
topic: "Plugin SDK Setup: Project Structure, package.json, TypeScript Config, and Dev Workflow"
type: procedure
keywords:
  - SDK setup
  - plugin project
  - plugin package.json
  - plugin TypeScript
  - plugin dev setup
  - plugin scaffold
related:
  - plugins/sdk-overview
  - plugins/building-plugins
source: plugins/sdk-setup.md
---

Setting up a new OpenClaw plugin project: directory structure, package.json, TypeScript config, and development workflow.

Reference for plugin packaging (`package.json` metadata), manifests (`openclaw.plugin.json`), setup entries, and config schemas.

**Looking for a walkthrough?** The how-to guides cover packaging in context: [Channel plugins](/plugins/sdk-channel-plugins#step-1-package-and-manifest) and [Provider plugins](/plugins/sdk-provider-plugins#step-1-package-and-manifest).

## Package metadata

Your `package.json` needs an `openclaw` field that tells the plugin system what your plugin provides:

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

If you publish the plugin externally on ClawHub, those `compat` and `build` fields are required. The canonical publish snippets live in `docs/snippets/plugin-publish/`.

### `openclaw` fields

<ParamField path="extensions" type="string[]">
  Entry point files (relative to package root).
</ParamField>
<ParamField path="setupEntry" type="string">
  Lightweight setup-only entry (optional).
</ParamField>
<ParamField path="channel" type="object">
  Channel catalog metadata for setup, picker, quickstart, and status surfaces.
</ParamField>
<ParamField path="providers" type="string[]">
  Provider ids registered by this plugin.
</ParamField>
<ParamField path="install" type="object">
  Install hints: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`.
</ParamField>
<ParamField path="startup" type="object">
  Startup behavior flags.
</ParamField>

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

`showConfigured` and `showInSetup` remain supported as legacy aliases. Prefer `exposure`.

### `openclaw.install`

`openclaw.install` is package metadata, not manifest metadata.

| Field                        | Type                 | What it means                                                                    |
| ---------------------------- | -------------------- | -------------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | Canonical npm spec for install/update flows.                                     |
| `localPath`                  | `string`             | Local development or bundled install path.                                       |
| `defaultChoice`              | `"npm"` \| `"local"` | Preferred install source when both are available.                                |
| `minHostVersion`             | `string`             | Minimum supported OpenClaw version in the form `>=x.y.z`.                        |
| `expectedIntegrity`          | `string`             | Expected npm dist integrity string, usually `sha512-...`, for pinned installs.   |
| `allowInvalidConfigRecovery` | `boolean`            | Lets bundled-plugin reinstall flows recover from specific stale-config failures. |

    Interactive onboarding also uses `openclaw.install` for install-on-demand surfaces. If your plugin exposes provider auth choices or channel setup/catalog metadata before runtime loads, onboarding can show that choice, prompt for npm vs local install, install or enable the plugin, then continue the selected flow. Npm onboarding choices require trusted catalog metadata with a registry `npmSpec`; exact versions and `expectedIntegrity` are optional pins. If `expectedIntegrity` is present, install/update flows enforce it. Keep the "what to show" metadata in `openclaw.plugin.json` and the "how to install it" metadata in `package.json`.

    If `minHostVersion` is set, install and manifest-registry loading both enforce it. Older hosts skip the plugin; invalid version strings are rejected.

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

    `allowInvalidConfigRecovery` is not a general bypass for broken configs. It is for narrow bundled-plugin recovery only, so reinstall/setup can repair known upgrade leftovers like a missing bundled plugin path or stale `channels.<id>` entry for that same plugin. If config is broken for unrelated reasons, install still fails closed and tells the operator to run `ope

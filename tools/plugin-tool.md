---
domain: tools
topic: "Plugin Tool: Dynamic Tool Registration"
type: concept
keywords:
  - plugin tool
  - tool registration
  - dynamic tools
  - tool plugin
  - custom tools
source: tools/plugin.md
---

Plugins extend OpenClaw with new capabilities: channels, model providers,
agent harnesses, tools, skills, speech, realtime transcription, realtime
voice, media-understanding, image generation, video generation, web fetch, web
search, and more. Some plugins are **core** (shipped with OpenClaw), others
are **external**. Most external plugins are published and discovered through
[ClawHub](/clawhub). Npm remains supported for direct installs and for a
temporary set of OpenClaw-owned plugin packages while that migration finishes.

## Quick start

For copy-paste install, list, uninstall, update, and publishing examples, see
[Manage plugins](/plugins/manage-plugins).

**See what is loaded**

```bash
    openclaw plugins list
    ```
  
**Install a plugin**

```bash
    # Search ClawHub plugins
    openclaw plugins search "calendar"

    # From ClawHub
    openclaw plugins install clawhub:openclaw-codex-app-server

    # From npm
    openclaw plugins install npm:@acme/openclaw-plugin
    openclaw plugins install npm-pack:./openclaw-plugin-1.2.3.tgz

    # From git
    openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0

    # From a local directory or archive
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

  
**Restart the Gateway**

```bash
    openclaw gateway restart
    ```

    Then configure under `plugins.entries.\<id\>.config` in your config file.

  
**Chat-native management**

In a running Gateway, owner-only `/plugins enable` and `/plugins disable`
    trigger the Gateway config reloader. The Gateway reloads plugin runtime
    surfaces in process, and new agent turns rebuild their tool list from the
    refreshed registry. `/plugins install` changes plugin source code, so the
    Gateway requests a restart instead of pretending the current process can
    safely reload already-imported modules.

  
**Verify the plugin**

```bash
    openclaw plugins inspect <plugin-id> --runtime --json

    # If the plugin registered a CLI root, run one command from that root.
    openclaw <plugin-command> --help
    ```

    Use `--runtime` when you need to prove registered tools, services, gateway
    methods, hooks, or plugin-owned CLI commands. Plain `inspect` is a cold
    manifest/registry check and intentionally avoids importing plugin runtime.

  
If you prefer chat-native control, enable `commands.plugins: true` and use:

```text
/plugin install clawhub:<package>
/plugin show <plugin-id>
/plugin enable <plugin-id>
```

The install path uses the same resolver as the CLI: local path/archive, explicit
`clawhub:<pkg>`, explicit `npm:<pkg>`, explicit `npm-pack:<path.tgz>`,
explicit `git:<repo>`, or bare package spec through npm.

If config is invalid, install normally fails closed and points you at
`openclaw doctor --fix`. The only recovery exception is a narrow bundled-plugin
reinstall path for plugins that opt into
`openclaw.install.allowInvalidConfigRecovery`.
During Gateway startup, invalid plugin config fails closed like any other invalid
config. Run `openclaw doctor --fix` to quarantine the bad plugin config by
disabling that plugin entry and removing its invalid config payload; the normal
config backup keeps the previous values.
When a channel config references a plugin that is no longer discoverable but the
same stale plugin id remains in plugin config or install records, Gateway startup
logs warnings and skips that channel instead of blocking every other channel.
Run `openclaw doctor --fix` to remove the stale channel/plugin entries; unknown
channel keys without stale-plugin evidence still fail validation so typos stay
visible.
If `plugins.enabled: false` is set, stale plugin references are treated as inert:
Gateway startup skips plugin discovery/load work and `openclaw doctor` preserves
the disabled plugin config instead of auto-removing it. Re-enable plugins before
running doctor cleanup if you want stale plugin ids removed.

Plugin dependency installation happens only during explicit install/update or
doctor repair flows. Gateway startup, config reload, and runtime inspection do
not run package managers or repair dependency trees. Local plugins must already
have their dependencies installed, while npm, git, and ClawHub plugins are
installed under OpenClaw's managed plugin roots. npm dependencies may be hoisted
within OpenClaw's managed npm root; install/update scans that managed root before
trust and uninstall removes npm-managed packages through npm. External plugins
and custom load paths must still be installed through `openclaw plugins install`.
Use `openclaw plugins list --json` to see the static `dependencyStatus` for each
visible plugin without importing runtime code or repairing dependencies.
See [Plugin dependency resolution](/plugins/dependency-resolution) for the
install-time lifecycle.

### Blocked plugin path ownership

If plugin diagnostics say
`blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`
and config validation follows with `plugin present but blocked`, OpenClaw found
plugin files owned by a different Unix user than the process that is loading
them. Keep the plugin config in place; fix the filesystem ownership or run
OpenClaw as the same user that owns the state directory.

For Docker installs, the official image runs as `node` (uid `1000`), so the
host bind-mounted OpenClaw config and workspace directories should normally be
owned by uid `1000`:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

If you intentionally run OpenClaw as root, repair the managed plugin root to
root ownership instead:

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
```

After fixing ownership, rerun `openclaw doctor --fix` or
`openclaw plugins registry --refresh` so the persisted plugin registry matches
the repaired files.

For npm installs, mutable selectors such as `latest` or a dist-tag are resolved
before installation and then pinned to the exact verified version in OpenClaw's
managed npm root. After npm finishes, OpenClaw verifies the installed
`package-lock.json` entry still matches the resolved version and integrity. If
npm writes different package metadata, the install fails and the managed package
is rolled back instead of accepting a different plugin artifact.
Managed npm roots also inherit OpenClaw's package-level npm `overrides`, so
security pins that protect the packaged host also apply to hoisted external
plugin dependencies.

Source checkouts are pnpm workspaces. If you clone OpenClaw to hack on bundled
plugins, run `pnpm install`; OpenClaw then loads bundled plugins from
`extensions/<id>` so edits and package-local dependencies are used directly.
Plain npm root installs are for packaged OpenClaw, not source checkout
development.

## Plugin types

OpenClaw recognizes two plugin formats:

| Format     | How it works                                                       | Examples                                               |
| ---------- | ------------------------------------------------------------------ | ------------------------------------------------------ |
| **Native** | `openclaw.plugin.json` + runtime module; executes in-process       | Official plugins, community npm packages               |
| **Bundle** | Codex/Claude/Cursor-compatible layout; mapped to OpenClaw features | `.codex-plugin/`, `.claude-plugin/`, `.cursor-plugin/` |

Both show up under `openclaw plugins list`. See [Plugin Bundles](/plugins/bundles) for bundle details.

If you are writing a native plugin, start with [Building Plugins](/plugins/building-plugins)
and the [Plugin SDK Overview](/plugins/sdk-overview).

## Package entrypoints

Native plugin npm packages must declare `openclaw.extensions` in `package.json`.
Each entry must stay inside the package directory and resolve to a readable
runtime file, or to a TypeScript source file with an inferred built JavaScript
peer such as `src/index.ts` to `dist/index.js`.
Packaged installs must ship that JavaScript runtime output. The TypeScript
source fallback is for source checkouts and local development paths, not for
npm packages installed into OpenClaw's managed plugin root.

Untracked directories dropped into the global extension root are treated as
local source checkouts and may load TypeScript entries directly. Directories
still named by an install record, including `installPath` or `sourcePath`, stay
managed and keep the compiled-output requirement even when the global scan sees
them. If you intentionally convert a managed install into an untracked local
checkout, remove the stale install record first with uninstall or doctor cleanup.

If a managed package warning says it `requires compiled runtime output for
TypeScript entry ...`, the package was published without the JavaScript files
OpenClaw needs at runtime. That is a plugin packaging issue, not a local config
problem. Update or reinstall the plugin after the publisher republishes compiled
JavaScript, or disable/uninstall that plugin until a fixed package is available.

Use `openclaw.runtimeExtensions` when published runtime files do not live at the
same paths as the source entries. When present, `runtimeExtensions` must contain
exactly one entry for every `extensions` entry. Mismatched lists fail install and
plugin discovery rather than silently falling back to source paths. If you also
publish `openclaw.setupEntry`, use `openclaw.runtimeSetupEntry` for its built
JavaScript peer; that file is required when declared.

```json
{
  "name": "@acme/openclaw-plugin",
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"]
  }
}
```

## Official plugins

### OpenClaw-owned npm packages during migration

ClawHub is the primary distribution path for most plugins. Current packaged
OpenClaw releases already bundle many official plugins, so those do not need
separate npm installs in normal setups. Until every OpenClaw-owned plugin has
migrated to ClawHub, OpenClaw still ships some `@openclaw/*` plugin packages on
npm for older/custom installs and direct npm workflows.

If npm reports an `@openclaw/*` plugin package as deprecated, that package
version is from an older external package train. Use the bundled plugin from
current OpenClaw or a local checkout until a newer npm package is published.

| Plugin          | Package                    | Docs                                       |
| --------------- | -------------------------- | ------------------------------------------ |
| Discord         | `@openclaw/discord`        | [Discord](/channels/discord)               |
| Feishu          | `@openclaw/feishu`         | [Feishu](/channels/feishu)                 |
| Matrix          | `@openclaw/matrix`         | [Matrix](/channels/matrix)                 |
| Mattermost      | `@openclaw/mattermost`     | [Mattermost](/channels/mattermost)         |
| Microsoft Teams | `@openclaw/msteams`        | [Microsoft Teams](/channels/msteams)       |
| Nextcloud Talk  | `@openclaw/nextcloud-talk` | [Nextcloud Talk](/channels/nextcloud-talk) |
| Nostr           | `@openclaw/nostr`          | [Nostr](/channels/nostr)                   |
| Synology Chat   | `@openclaw/synology-chat`  | [Synology Chat](/channels/synology-chat)   |
| Tlon            | `@openclaw/tlon`           | [Tlon](/channels/tlon)                     |
| WhatsApp        | `@openclaw/whatsapp`       | [WhatsApp](/channels/whatsapp)             |
| Zalo            | `@openclaw/zalo`           | [Zalo](/channels/zalo)                     |
| Zalo Personal   | `@openclaw/zalouser`       | [Zalo Personal](/plugins/zalouser)         |

### Core (shipped with OpenClaw)

### Model providers (enabled by default)

`anthropic`, `byteplus`, `cloudflare-ai-gateway`, `github-copilot`, `google`,
    `huggingface`, `kilocode`, `kimi-coding`, `minimax`, `mistral`, `qwen`,
    `moonshot`, `nvidia`, `openai`, `opencode`, `opencode-go`, `openrouter`,
    `qianfan`, `synthetic`, `together`, `venice`,
    `vercel-ai-gateway`, `volcengine`, `xiaomi`, `zai`
  ### Memory plugins

- `memory-core` - bundled memory search (default via `plugins.slots.memory`)
    - `memory-lancedb` - LanceDB-backed long-term memory with auto-recall/capture (set `plugins.slots.memory = "memory-lancedb"`)

    See [Memory LanceDB](/plugins/memory-lancedb) for OpenAI-compatible
    embedding setup, Ollama examples, recall limits, and troubleshooting.

  ### Speech providers (enabled by default)

`elevenlabs`, `microsoft`
  ### Other

- `browser` - bundled browser plugin for the browser tool, `openclaw browser` CLI, `browser.request` gateway method, browser runtime, and default browser control service (enabled by default; disable before replacing it)
    - `copilot-proxy` - VS Code Copilot Proxy bridge (disabled by default)

  Looking for third-party plugins? See [ClawHub](/clawhub).

## Configuration

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-plugin"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

| Field              | Description                                               |
| ------------------ | --------------------------------------------------------- |
| `enabled`          | Master toggle (default: `true`)                           |
| `allow`            | Plugin allowlist (optional)                               |
| `bundledDiscovery` | Bundled plugin discovery mode (`allowlist` by default)    |
| `deny`             | Plugin denylist (optional; deny wins)                     |
| `load.paths`       | Extra plugin files/directories                            |
| `slots`            | Exclusive slot selectors (e.g. `memory`, `contextEngine`) |
| `entries.\<id\>`   | Per-plugin toggles + config                               |

`plugins.allow` is exclusive. When it is non-empty, 
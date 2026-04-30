---
domain: tools
topic: "Plugin Tool: Agent Interaction with Plugin-Provided Tools and Capabilities"
type: concept
keywords:
  - plugin tool
  - plugin interaction
  - plugin-provided tools
  - agent plugins
related:
  - plugins/plugin-architecture
  - plugins/building-plugins
  - tools/tools-overview
source: tools/plugin.md
---

The plugin tool allows the agent to interact with installed plugins. Covers how agents call plugin-provided tools.

Plugins extend OpenClaw with new capabilities: channels, model providers,
agent harnesses, tools, skills, speech, realtime transcription, realtime
voice, media-understanding, image generation, video generation, web fetch, web
search, and more. Some plugins are **core** (shipped with OpenClaw), others
are **external**. Most external plugins are published and discovered through
[ClawHub](/tools/clawhub). Npm remains supported for direct installs and for a
temporary set of OpenClaw-owned plugin packages while that migration finishes.

## Quick start

    ```bash
    openclaw plugins list
    ```

    ```bash
    # From npm
    openclaw plugins install npm:@acme/openclaw-plugin

    # From a local directory or archive
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

    ```bash
    openclaw gateway restart
    ```

    Then configure under `plugins.entries.\<id\>.config` in your config file.

If you prefer chat-native control, enable `commands.plugins: true` and use:

```text
/plugin install clawhub:<package>
/plugin show <plugin-id>
/plugin enable <plugin-id>
```

The install path uses the same resolver as the CLI: local path/archive, explicit
`clawhub:<pkg>`, explicit `npm:<pkg>`, or bare package spec (ClawHub first, then
npm fallback).

If config is invalid, install normally fails closed and points you at
`openclaw doctor --fix`. The only recovery exception is a narrow bundled-plugin
reinstall path for plugins that opt into
`openclaw.install.allowInvalidConfigRecovery`.
During Gateway startup, invalid config for one plugin is isolated to that plugin:
startup logs the `plugins.entries.<id>.config` issue, skips that plugin during
load, and keeps other plugins and channels online. Run `openclaw doctor --fix`
to quarantine the bad plugin config by disabling that plugin entry and removing
its invalid config payload; the normal config backup keeps the previous values.
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

Packaged OpenClaw installs do not eagerly install every bundled plugin's
runtime dependency tree. When a bundled OpenClaw-owned plugin is active from
plugin config, legacy channel config, or a default-enabled manifest, startup
repairs only that plugin's declared runtime dependencies before importing it.
Persisted channel auth state alone does not activate a bundled channel for
Gateway startup runtime-dependency repair.
Explicit disablement still wins: `plugins.entries.<id>.enabled: false`,
`plugins.deny`, `plugins.enabled: false`, and `channels.<id>.enabled: false`
prevent automatic bundled runtime-dependency repair for that plugin/channel.
A non-empty `plugins.allow` also bounds default-enabled bundled runtime-dependency
repair; explicit bundled channel enablement (`channels.<id>.enabled: true`) can
still repair that channel's plugin dependencies.
External plugins and custom load paths must still be installed through
`openclaw plugins install`.

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

Use `openclaw.runtimeExtensions` when published runtime files do not live at the
same paths as the source entries. When present, `runtimeExtensions` must contain
exactly one entry for every `extensions` entry. Mismatched lists fail install and
plugin discovery rather than silently falling back to source paths.

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
| BlueBubbles     | `@openclaw/bluebubbles`    | [BlueBubbles](/channels/bluebubbles)       |
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

    `anthropic`, `byteplus`, `cloudflare-ai-gateway`, `github-copilot`, `google`,
    `huggingface`, `kilocode`, `kimi-coding`, `minimax`, `mistral`, `qwen`,
    `moonshot`, `nvidia`, `openai`, `opencode`, `opencode-go`, `openrouter`,
    `qianfan`, `synthetic`, `together`, `venice`,
    `vercel-ai-gateway`, `volcengine`, `xiaomi`, `zai`

    - `memory-core` — bundled memory search (default via `plugins.slots.memory`)
    - `memory-lancedb` — install-on-demand long-term memory with auto-recall/capture (set `plugins.slots.memory = "memory-lancedb"`)

    See [Memory LanceDB](/plugins/memory-lancedb) for OpenAI-compatible
    embedding setup, Ollama examples, recall limits, and troubleshooting.

    `elevenlabs`, `microsoft`

    - `browser` — bundled browser plugin for the browser tool, `openclaw browser` CLI, `browser.request` gateway method, browser runtime, and default browser control service (enabled by default; disable before replacing it)
    - `copilot-proxy` — VS Code Copilot Proxy bridge (disabled by default)

Looking for third-party plugins? See [Community Plugins](/plugins/community).

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

| Field            | Description                                               |
| ---------------- | --------------------------------------------------------- |
| `enabled`        | Master toggle (default: `true`)

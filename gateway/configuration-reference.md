---
domain: gateway
topic: "Complete Gateway Configuration Reference"
type: reference
keywords:
  - openclaw.json
  - configuration reference
  - agents.defaults
  - channels config
  - models config
  - gateway config
  - plugins config
  - memory config
source: gateway/configuration-reference.md
related:
  - gateway/configuration-overview
  - gateway/config-agents
  - gateway/config-channels
---

Core config reference for `~/.openclaw/openclaw.json`. For a task-oriented overview, see [Configuration](/gateway/configuration).

Covers the main OpenClaw config surfaces and links out when a subsystem has its own deeper reference. Channel- and plugin-owned command catalogs and deep memory/QMD knobs live on their own pages rather than on this one.

Code truth:

- `openclaw config schema` prints the live JSON Schema used for validation and Control UI, with bundled/plugin/channel metadata merged in when available
- `config.schema.lookup` returns one path-scoped schema node for drill-down tooling
- `pnpm config:docs:check` / `pnpm config:docs:gen` validate the config-doc baseline hash against the current schema surface

Agent lookup path: use the `gateway` tool action `config.schema.lookup` for
exact field-level docs and constraints before edits. Use
[Configuration](/gateway/configuration) for task-oriented guidance and this page
for the broader field map, defaults, and links to subsystem references.

Dedicated deep references:

- [Memory configuration reference](/reference/memory-config) for `agents.defaults.memorySearch.*`, `memory.qmd.*`, `memory.citations`, and dreaming config under `plugins.entries.memory-core.config.dreaming`
- [Slash commands](/tools/slash-commands) for the current built-in + bundled command catalog
- owning channel/plugin pages for channel-specific command surfaces

Config format is **JSON5** (comments + trailing commas allowed). All fields are optional - OpenClaw uses safe defaults when omitted.

---

## Channels

Per-channel config keys moved to a dedicated page - see
[Configuration - channels](/gateway/config-channels) for `channels.*`,
including Slack, Discord, Telegram, WhatsApp, Matrix, iMessage, and other
bundled channels (auth, access control, multi-account, mention gating).

## Agent defaults, multi-agent, sessions, and messages

Moved to a dedicated page - see
[Configuration - agents](/gateway/config-agents) for:

- `agents.defaults.*` (workspace, model, thinking, heartbeat, memory, media, skills, sandbox)
- `multiAgent.*` (multi-agent routing and bindings)
- `session.*` (session lifecycle, compaction, pruning)
- `messages.*` (message delivery, TTS, markdown rendering)
- `talk.*` (Talk mode)
  - `talk.consultThinkingLevel`: thinking level override for the full OpenClaw agent run behind Control UI Talk realtime consults
  - `talk.consultFastMode`: one-shot fast-mode override for Control UI Talk realtime consults
  - `talk.speechLocale`: optional BCP 47 locale id for Talk speech recognition on iOS/macOS
  - `talk.silenceTimeoutMs`: when unset, Talk keeps the platform default pause window before sending the transcript (`700 ms on macOS and Android, 900 ms on iOS`)

## Tools and custom providers

Tool policy, experimental toggles, provider-backed tool config, and custom
provider / base-URL setup moved to a dedicated page - see
[Configuration - tools and custom providers](/gateway/config-tools).

## Models

Provider definitions, model allowlists, and custom provider setup live in
[Configuration - tools and custom providers](/gateway/config-tools#custom-providers-and-base-urls).
The `models` root also owns global model-catalog behavior.

```json5
{
  models: {
    // Optional. Default: true. Requires a Gateway restart when changed.
    pricing: { enabled: false },
  },
}
```

- `models.mode`: provider catalog behavior (`merge` or `replace`).
- `models.providers`: custom provider map keyed by provider id.
- `models.providers.*.localService`: optional on-demand process manager for
  local model servers. OpenClaw probes the configured health endpoint, starts
  the absolute `command` when needed, waits for readiness, then sends the model
  request. See [Local model services](/gateway/local-model-services).
- `models.pricing.enabled`: controls the background pricing bootstrap that
  starts after sidecars and channels reach the Gateway ready path. When `false`,
  the Gateway skips OpenRouter and LiteLLM pricing-catalog fetches; configured
  `models.providers.*.models[].cost` values still work for local cost estimates.

## MCP

OpenClaw-managed MCP server definitions live under `mcp.servers` and are
consumed by embedded Pi and other runtime adapters. The `openclaw mcp list`,
`show`, `set`, and `unset` commands manage this block without connecting to the
target server during config edits.

```json5
{
  mcp: {
    // Optional. Default: 600000 ms (10 minutes). Set 0 to disable idle eviction.
    sessionIdleTtlMs: 600000,
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
      },
    },
  },
}
```

- `mcp.servers`: named stdio or remote MCP server definitions for runtimes that
  expose configured MCP tools.
  Remote entries use `transport: "streamable-http"` or `transport: "sse"`;
  `type: "http"` is a CLI-native alias that `openclaw mcp set` and
  `openclaw doctor --fix` normalize into the canonical `transport` field.
- `mcp.sessionIdleTtlMs`: idle TTL for session-scoped bundled MCP runtimes.
  One-shot embedded runs request run-end cleanup; this TTL is the backstop for
  long-lived sessions and future callers.
- Changes under `mcp.*` hot-apply by disposing cached session MCP runtimes.
  The next tool discovery/use recreates them from the new config, so removed
  `mcp.servers` entries are reaped immediately instead of waiting for idle TTL.

See [MCP](/cli/mcp#openclaw-as-an-mcp-client-registry) and
[CLI backends](/gateway/cli-backends#bundle-mcp-overlays) for runtime behavior.

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: optional allowlist for bundled skills only (managed/workspace skills unaffected).
- `load.extraDirs`: extra shared skill roots (lowest precedence).
- `load.allowSymlinkTargets`: trusted real target roots that skill symlinks may
  resolve into when the link lives outside its configured source root.
- `install.preferBrew`: when true, prefer Homebrew installers when `brew` is
  available before falling back to other installer kinds.
- `install.nodeManager`: node installer preference for `metadata.openclaw.install`
  specs (`npm` | `pnpm` | `yarn` | `bun`).
- `install.allowUploadedArchives`: allow trusted `operator.admin` Gateway
  clients to install private zip archives staged through `skills.upload.*`
  (default: false). This only enables the uploaded-archive path; normal ClawHub
  installs do not require it.
- `entries.<skillKey>.enabled: false` disables a skill even if bundled/installed.
- `entries.<skillKey>.apiKey`: convenience for skills declaring a primary env var (plaintext string or SecretRef object).

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    bundledDiscovery: "allowlist",
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- Loaded from `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions`, plus `plugins.load.paths`.
- Discovery accepts native OpenClaw plugins plus compatible Codex bundles and Claude bundles, including manifestless Claude default-layout bundles.
- **Config changes require a gateway restart.**
- `allow`: optional allowlist (only listed plugins load). `deny` wins.
- `bundledDiscovery`: defaults to `"allowlist"` for new configs, so a non-empty
  `plugins.allow` also gates bundled provider plugins, including web-search
  runtime providers. Doctor writes `"compat"` for migrated legacy allowlist
  configs to preserve existing bundled provider behavior until you opt in.
- `plugins.entries.<id>.apiKey`: plugin-level API key convenience field (when supported by the plugin).
- `plugins.entries.<id>.env`: plugin-scoped env var map.
- `plugins.entries.<id>.hooks.allowPromptInjection`: when `false`, core blocks `before_prompt_build` and ignores prompt-mutating fields from legacy `before_agent_start`, while preserving legacy `modelOverride` and `providerOverride`. Applies to native plugin hooks and supported bundle-provided hook directories.
- `plugins.entries.<id>.hooks.allowConversationAccess`: when `true`, trusted non-bundled plugins may read raw conversation content from typed hooks such as `llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize`, and `agent_end`.
- `plugins.entries.<id>.subagent.allowModelOverride`: explicitly trust this plugin to request per-run `provider` and `model` overrides for background subagent runs.
- `plugins.entries.<id>.subagent.allowedModels`: optional allowlist of canonical `provider/model` targets for trusted subagent overrides. Use `"*"` only when you intentionally want to allow any model.
- `plugins.entries.<id>.llm.allowModelOverride`: explicitly trust this plugin to request model overrides for `api.runtime.llm.complete`.
- `plugins.entries.<id>.llm.allowedModels`: optional allowlist of canonical `provider/model` targets for trusted plugin LLM completion overrides. Use `"*"` only when you intentionally want to allow any model.
- `plugins.entries.<id>.llm.allowAgentIdOverride`: explicitly trust this plugin to run `api.runtime.llm.complete` against a non-default agent id.
- `plugins.entries.<id>.config`: plugin-defined config object (validated by native OpenClaw plugin schema when available).
- Channel plugin account/runtime settings live under `channels.<id>` and should be described by the owning plugin's manifest `channelConfigs` metadata, not by a central OpenClaw option registry.

### Codex harness plugin config

The bundled `codex` plugin owns native Codex app-server harness settings under
`plugins.entries.codex.config`. See
[Codex harness reference](/plugins/codex-harness-reference) for the full config
surface and [Codex harness](/plugins/codex-harness) for the runtime model.

`codexPlugins` applies only to sessions that select the native Codex harness.
It does not enable Codex plugins for Pi, normal OpenAI provider runs, ACP
conversation bindings, or any non-Codex harness.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: enables native Codex
  plugin/app support for the Codex harness. Default: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  default destructive-action policy for migrated plugin app elicitations.
  Default: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: enables a
  migrated plugin entry when global `codexPlugins.enabled` is also true.
  Default: `true` for explicit entries.
- `plugins.entries.code

<!-- See source for complete reference -->

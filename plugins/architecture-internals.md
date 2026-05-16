---
domain: plugins
topic: "Plugin Architecture Internals"
type: concept
keywords:
  - plugin internals
  - plugin loader
  - plugin registry
  - plugin resolution
  - manifest loading
source: plugins/architecture-internals.md
---

For the public capability model, plugin shapes, and ownership/execution
contracts, see [Plugin architecture](/plugins/architecture). This page is the
reference for the internal mechanics: load pipeline, registry, runtime hooks,
Gateway HTTP routes, import paths, and schema tables.

## Load pipeline

At startup, OpenClaw does roughly this:

1. discover candidate plugin roots
2. read native or compatible bundle manifests and package metadata
3. reject unsafe candidates
4. normalize plugin config (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decide enablement for each candidate
6. load enabled native modules: built bundled modules use a native loader;
   third-party local source TypeScript uses the emergency Jiti fallback
7. call native `register(api)` hooks and collect registrations into the plugin registry
8. expose the registry to commands/runtime surfaces

> **Note:** `activate` is a legacy alias for `register` — the loader resolves whichever is present (`def.register ?? def.activate`) and calls it at the same point. All bundled plugins use `register`; prefer `register` for new plugins.


The safety gates happen **before** runtime execution. Candidates are blocked
when the entry escapes the plugin root, the path is world-writable, or path
ownership looks suspicious for non-bundled plugins.

Blocked candidates remain tied to their plugin id for diagnostics. If config
still references that id, validation reports the plugin as present but blocked
and points back to the path-safety warning instead of treating the config entry
as stale.

### Manifest-first behavior

The manifest is the control-plane source of truth. OpenClaw uses it to:

- identify the plugin
- discover declared channels/skills/config schema or bundle capabilities
- validate `plugins.entries.<id>.config`
- augment Control UI labels/placeholders
- show install/catalog metadata
- preserve cheap activation and setup descriptors without loading plugin runtime

For native plugins, the runtime module is the data-plane part. It registers
actual behavior such as hooks, tools, commands, or provider flows.

Optional manifest `activation` and `setup` blocks stay on the control plane.
They are metadata-only descriptors for activation planning and setup discovery;
they do not replace runtime registration, `register(...)`, or `setupEntry`.
The first live activation consumers now use manifest command, channel, and provider hints
to narrow plugin loading before broader registry materialization:

- CLI loading narrows to plugins that own the requested primary command
- channel setup/plugin resolution narrows to plugins that own the requested
  channel id
- explicit provider setup/runtime resolution narrows to plugins that own the
  requested provider id
- Gateway startup planning uses `activation.onStartup` for explicit startup
  imports and startup opt-outs; plugins without startup metadata load only
  through narrower activation triggers

Request-time runtime preloads that ask for the broad `all` scope still derive an
explicit effective plugin id set from config, startup planning, configured
channels, slots, and auto-enable rules. If that derived set is empty, OpenClaw
loads an empty runtime registry instead of widening to every discoverable
plugin.

The activation planner exposes both an ids-only API for existing callers and a
plan API for new diagnostics. Plan entries report why a plugin was selected,
separating explicit `activation.*` planner hints from manifest ownership
fallback such as `providers`, `channels`, `commandAliases`, `setup.providers`,
`contracts.tools`, and hooks. That reason split is the compatibility boundary:
existing plugin metadata keeps working, while new code can detect broad hints
or fallback behavior without changing runtime loading semantics.

Setup discovery now prefers descriptor-owned ids such as `setup.providers` and
`setup.cliBackends` to narrow candidate plugins before it falls back to
`setup-api` for plugins that still need setup-time runtime hooks. Provider
setup lists use manifest `providerAuthChoices`, descriptor-derived setup
choices, and install-catalog metadata without loading provider runtime. Explicit
`setup.requiresRuntime: false` is a descriptor-only cutoff; omitted
`requiresRuntime` keeps the legacy setup-api fallback for compatibility. If more
than one discovered plugin claims the same normalized setup provider or CLI
backend id, setup lookup refuses the ambiguous owner instead of relying on
discovery order. When setup runtime does execute, registry diagnostics report
drift between `setup.providers` / `setup.cliBackends` and the providers or CLI
backends registered by setup-api without blocking legacy plugins.

### Plugin cache boundary

OpenClaw does not cache plugin discovery results or direct manifest registry
data behind wall-clock windows. Installs, manifest edits, and load-path changes
must become visible on the next explicit metadata read or snapshot rebuild.
The manifest file parser may keep a bounded file-signature cache keyed by the
opened manifest path, inode, size, and timestamps; that cache only avoids
re-parsing unchanged bytes and must not cache discovery, registry, owner, or
policy answers.

The safe metadata fast path is explicit object ownership, not a hidden cache.
Gateway startup hot paths should pass the current `PluginMetadataSnapshot`, the
derived `PluginLookUpTable`, or an explicit manifest registry through the call
chain. Config validation, startup auto-enable, plugin bootstrap, and provider
selection can reuse those objects while they represent the current config and
plugin inventory. Setup lookup still reconstructs manifest metadata on demand
unless the specific setup path receives an explicit manifest registry; keep that
as a cold-path fallback rather than adding hidden lookup caches. When the input
changes, rebuild and replace the snapshot instead of mutating it or keeping
historical copies.
Views over the active plugin registry and bundled channel bootstrap helpers
should be recomputed from the current registry/root. Short-lived maps are fine
inside one call to dedupe work or guard reentry; they must not become process
metadata caches.

For plugin loading, the persistent cache layer is runtime loading. It may reuse
loader state when code or installed artifacts are actually loaded, such as:

- `PluginLoaderCacheState` and compatible active runtime registries
- jiti/module caches and public-surface loader caches used to avoid importing
  the same runtime surface repeatedly
- filesystem caches for installed plugin artifacts
- short-lived per-call maps for path normalization or duplicate resolution

Those caches are data-plane implementation details. They must not answer
control-plane questions such as "which plugin owns this provider?" unless the
caller deliberately asked for runtime loading.

Do not add persistent or wall-clock caches for:

- discovery results
- direct manifest registries
- manifest registries reconstructed from the installed plugin index
- provider owner lookup, model suppression, provider policy, or public-artifact
  metadata
- any other manifest-derived answer where a changed manifest, installed index,
  or load path should be visible on the next metadata read

Callers that rebuild manifest metadata from the persisted installed plugin
index reconstruct that registry on demand. The installed index is durable
source-plane state; it is not a hidden in-process metadata cache.

## Registry model

Loaded plugins do not directly mutate random core globals. They register into a
central plugin registry.

The registry tracks:

- plugin records (identity, source, origin, status, diagnostics)
- tools
- legacy hooks and typed hooks
- channels
- providers
- gateway RPC handlers
- HTTP routes
- CLI registrars
- background services
- plugin-owned commands

Core features then read from that registry instead of talking to plugin modules
directly. This keeps loading one-way:

- plugin module -> registry registration
- core runtime -> registry consumption

That separation matters for maintainability. It means most core surfaces only
need one integration point: "read the registry", not "special-case every plugin
module".

## Conversation binding callbacks

Plugins that bind a conversation can react when an approval is resolved.

Use `api.onConversationBindingResolved(...)` to receive a callback after a bind
request is approved or denied:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Callback payload fields:

- `status`: `"approved"` or `"denied"`
- `decision`: `"allow-once"`, `"allow-always"`, or `"deny"`
- `binding`: the resolved binding for approved requests
- `request`: the original request summary, detach hint, sender id, and
  conversation metadata

This callback is notification-only. It does not change who is allowed to bind a
conversation, and it runs after core approval handling finishes.

## Provider runtime hooks

Provider plugins have three layers:

- **Manifest metadata** for cheap pre-runtime lookup:
  `setup.providers[].envVars`, deprecated compatibility `providerAuthEnvVars`,
  `providerAuthAliases`, `providerAuthChoices`, and `channelEnvVars`.
- **Config-time hooks**: `catalog` (legacy `discovery`) plus
  `applyConfigDefaults`.
- **Runtime hooks**: 40+ optional hooks covering auth, model resolution,
  stream wrapping, thinking levels, replay policy, and usage endpoints. See
  the full list under [Hook order and usage](#hook-order-and-usage).

OpenClaw still owns the generic agent loop, failover, transcript handling, and
tool policy. These hooks are the extension surface for provider-specific
behavior without needing a whole custom inference transport.

Use manifest `setup.providers[].envVars` when the provider has env-based
credentials that generic auth/status/model-picker paths should see without
loading plugin runtime. Deprecated `providerAuthEnvVars` is still read by the
compatibility adapter during the deprecation window, and non-bundled plugins
that use it receive a manifest diagnostic. Use manifest `providerAuthAliases`
when one provider id should reuse another provider id's env vars, auth profiles,
config-backed auth, and API-key onboarding choice. Use manifest
`providerAuthChoices` when onboarding/auth-choice CLI surfaces should know the
provider's choice id, group labels, and simple one-flag auth wiring without
loading provider runtime. Keep provider runtime
`envVars` for operator-facing hints such as onboarding labels or OAuth
client-id/client-secret setup vars.

Use manifest `channelEnvVars` when a channel has env-driven auth or setup that
generic shell-env fallback, config/status checks, or setup prompts should see
without loading channel runtime.

### Hook order and usage

For model/provider plugins, OpenClaw calls hooks in this rough order.
The "When to use" column is the quick decision guide.
Compatibility-only provider fields that OpenClaw no longer calls, such as
`ProviderPlugin.capabilities` and `suppressBuiltInModel`, are intentionally not
listed here.

| #   | Hook                              | What it does                                                                                                   | When to use                                                                                                                                   |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publish provider config into `models.providers` during `models.json` generation                                | Provider owns a catalog or base URL defaults                                                                                                  |
| 2   | `applyConfigDefaults`             | Apply provider-owned global config defaults during config materialization                                      | Defaults depend on auth mode, env, or provider model-family semantics                                                                         |
| --  | _(built-in model lookup)_         | OpenClaw tries the normal registry/catalog path first                                                          | _(not a plugin hook)_                                                                                                                         |
| 3   | `normalizeModelId`                | Normalize legacy or preview model-id aliases before lookup                                                     | Provider owns alias cleanup before canonical model resolution                                                                                 |
| 4   | `normalizeTransport`              | Normalize provider-family `api` / `baseUrl` before generic model assembly                                      | Provider owns transport cleanup for custom provider ids in the same transport family                                                          |
| 5   | `normalizeConfig`                 | Normalize `models.providers.<id>` before runtime/provider resolution                                           | Provider needs config cleanup that should live with the plugin; bundled Google-family helpers also backstop supported Google config entries   |
| 6   | `applyNativeStreamingUsageCompat` | Apply native streaming-usage compat rewrites to config providers                                
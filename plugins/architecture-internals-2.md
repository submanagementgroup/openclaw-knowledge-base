---
domain: plugins
topic: "Plugin Internals: Message Schemas, Context Engine Plugins, and Capability Checklist"
type: reference
keywords:
  - message tool schemas
  - context engine plugins
  - capability checklist
  - plugin catalog
related:
  - plugins/architecture-internals
  - plugins/sdk-overview
source: plugins/architecture-internals.md
---

Plugin architecture internals continued: message tool schemas, context engine plugins, and capability checklist.

stry through the call
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
- runtime dependency mirrors and filesystem caches for installed plugin
  artifacts
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
| 2   | `applyConfigDefaults`             | Apply provide

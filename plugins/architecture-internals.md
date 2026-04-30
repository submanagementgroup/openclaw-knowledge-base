---
domain: plugins
topic: "Plugin Architecture Internals: Load Pipeline, Registry, Provider Hooks, and HTTP Routes"
type: reference
keywords:
  - plugin internals
  - plugin load pipeline
  - plugin registry
  - provider hooks
  - gateway routes
related:
  - plugins/plugin-architecture
  - plugins/sdk-overview
source: plugins/architecture-internals.md
---

Deep dive into OpenClaw plugin internals: load pipeline, registry model, provider runtime hooks, and gateway HTTP routes.

## Plugin Load Pipeline

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
   unbuilt native plugins use jiti
7. call native `register(api)` hooks and collect registrations into the plugin registry
8. expose the registry to commands/runtime surfaces

`activate` is a legacy alias for `register` — the loader resolves whichever is present (`def.register ?? def.activate`) and calls it at the same point. All bundled plugins use `register`; prefer `register` for new plugins.

The safety gates happen **before** runtime execution. Candidates are blocked
when the entry escapes the plugin root, the path is world-writable, or path
ownership looks suspicious for non-bundled plugins.

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
  imports and startup opt-outs; every plugin should declare it as OpenClaw
  moves away from implicit startup imports, while plugins without static
  capability metadata and without `activation.onStartup` still use the
  deprecated implicit startup sidecar fallback for compatibility

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
derived `PluginLookUpTable`, or an explicit manifest regi

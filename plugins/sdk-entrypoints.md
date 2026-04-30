---
domain: plugins
topic: "Plugin Entrypoints: Registration with the Gateway at Load Time"
type: reference
keywords:
  - plugin entrypoints
  - plugin registration
  - load time
  - plugin bootstrap
  - entrypoint
related:
  - plugins/sdk-overview
  - plugins/plugin-manifest
source: plugins/sdk-entrypoints.md
---

Plugin entrypoints: how plugins register themselves with the Gateway at load time.

Every plugin exports a default entry object. The SDK provides three helpers for
creating them.

For installed plugins, `package.json` should point runtime loading at built
JavaScript when available:

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

`extensions` and `setupEntry` remain valid source entries for workspace and git
checkout development. `runtimeExtensions` and `runtimeSetupEntry` are preferred
when OpenClaw loads an installed package and let npm packages avoid runtime
TypeScript compilation. If an installed package only declares a TypeScript
source entry, OpenClaw will use a matching built `dist/*.js` peer when one
exists, then fall back to the TypeScript source.

All entry paths must stay inside the plugin package directory. Runtime entries
and inferred built JavaScript peers do not make an escaping `extensions` or
`setupEntry` source path valid.

  **Looking for a walkthrough?** See [Channel Plugins](/plugins/sdk-channel-plugins)
  or [Provider Plugins](/plugins/sdk-provider-plugins) for step-by-step guides.

## `definePluginEntry`

**Import:** `openclaw/plugin-sdk/plugin-entry`

For provider plugins, tool plugins, hook plugins, and anything that is **not**
a messaging channel.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
  },
});
```

| Field          | Type                                                             | Required | Default             |
| -------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`           | `string`                                                         | Yes      | —                   |
| `name`         | `string`                                                         | Yes      | —                   |
| `description`  | `string`                                                         | Yes      | —                   |
| `kind`         | `string`                                                         | No       | —                   |
| `configSchema` | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | No       | Empty object schema |
| `register`     | `(api: OpenClawPluginApi) => void`                               | Yes      | —                   |

- `id` must match your `openclaw.plugin.json` manifest.
- `kind` is for exclusive slots: `"memory"` or `"context-engine"`.
- `configSchema` can be a function for lazy evaluation.
- OpenClaw resolves and memoizes that schema on first access, so expensive schema
  builders only run once.

## `defineChannelPluginEntry`

**Import:** `openclaw/plugin-sdk/channel-core`

Wraps `definePluginEntry` with channel-specific wiring. Automatically calls
`api.registerChannel({ plugin })`, exposes an optional root-help CLI metadata
seam, and gates `registerFull` on registration mode.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| Field                 | Type                                                             | Required | Default             |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | Yes      | —                   |
| `name`                | `string`                                                         | Yes      | —                   |
| `description`         | `string`                                                         | Yes      | —                   |
| `plugin`              | `ChannelPlugin`                                                  | Yes      | —                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | No       | Empty object schema |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | No       | —                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | No       | —                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | No       | —                   |

- `setRuntime` is called during registration so you can store the runtime reference
  (typically via `createPluginRuntimeStore`). It is skipped during CLI metadata
  capture.
- `registerCliMetadata` runs during `api.registrationMode === "cli-metadata"`,
  `api.registrationMode === "discovery"`, and
  `api.registrationMode === "full"`.
  Use it as the canonical place for channel-owned CLI descriptors so root help
  stays non-activating, discovery snapshots include static command metadata, and
  normal CLI command registration remains compatible with full plugin loads.
- Discovery registration is non-activating, not import-free. OpenClaw may
  evaluate the trusted plugin entry and channel plugin module to build the
  snapshot, so keep top-level imports side-effect-free and put sockets,
  clients, workers, and services behind `"full"`-only paths.
- `registerFull` only runs when `api.registrationMode === "full"`. It is skipped
  during setup-only loading.
- Like `definePluginEntry`, `configSchema` can be a lazy factory and OpenClaw
  memoizes the resolved schema on first access.
- For plugin-owned root CLI commands, prefer `api.registerCli(..., { descriptors: [...] })`
  when you want the command to stay lazy-loaded without disappearing from the
  root CLI parse tree. For channel plugins, prefer registering those descriptors
  from `registerCliMetadata(...)` and keep `registerFull(...)` focused on runtime-only work.
- If `registerFull(...)` also registers gateway RPC methods, keep them on a
  plugin-specific prefix. Reserved core admin namespaces (`config.*`,
  `exec.approvals.*`, `wizard.*`, `update.*`) are always coerced to
  `operator.admin`.

## `defineSetupPluginEntry`

**Import:** `openclaw/plugin-sdk/channel-core`

For the lightweight `setup-entry.ts` file. Returns just `{ plugin }` with no
runtime or CLI wiring.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

OpenClaw loads this instead of the full entry when a channel is disabled,
unconfigured, or when deferred loading is enabled. See
[Setup and Config](/plugins/sdk-setup#setup-entry) for when this matters.

In practice, pair `defineSetupPluginEntry(...)` with the narrow setup helper
families:

- `openclaw/plugin-sdk/setup-runtime` for runtime-safe setup helpers such as
  import-safe setup patch adapters, lookup-note output,
  `promptResolvedAllowFrom`, `splitSetupEntries`, and delegated setup proxies
- `openclaw/plugin-sdk/channel-setup` for optional-install setup surfaces
- `openclaw/plugin-sdk/setup-tools` for setup/install CLI/archive/docs helpers

Keep heavy SDKs, CLI registration, and long-lived runtime services in the full
entry.

Bundled workspace channels that split setup and runtime surfaces can use
`defineBundledChannelSetupEntry(...)` from
`openclaw/plugin-sdk/channel-entry-contract` instead. That contract lets the
setup entry keep setup-safe plugin/secrets exports while still exposing a
runtime setter:

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
});
```

Use that bundled contract only when setup flows truly need a lightweight runtime
setter before the full channel entry loads.

## Registration mode

`api.registrationMode` tells your plugin how it was loaded:

| Mode              | When                              | What to register                                                                                                        |
| ----------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `"full"`          | Normal gateway startup            | Everything

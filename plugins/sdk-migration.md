---
domain: plugins
topic: "Plugin SDK Migration Guide: Breaking Changes, Import Paths, and Deprecation Timeline"
type: procedure
keywords:
  - SDK migration
  - plugin migration
  - import paths
  - deprecation
  - API changes
  - migration guide
related:
  - plugins/sdk-overview
  - plugins/building-plugins
source: plugins/sdk-migration.md
---

Migration guide for OpenClaw plugin SDK breaking changes. Covers import path changes, deprecated APIs, and the migration timeline.

OpenClaw has moved from a broad backwards-compatibility layer to a modern plugin
architecture with focused, documented imports. If your plugin was built before
the new architecture, this guide helps you migrate.

## What is changing

The old plugin system provided two wide-open surfaces that let plugins import
anything they needed from a single entry point:

- **`openclaw/plugin-sdk/compat`** — a single import that re-exported dozens of
  helpers. It was introduced to keep older hook-based plugins working while the
  new plugin architecture was being built.
- **`openclaw/plugin-sdk/infra-runtime`** — a broad runtime helper barrel that
  mixed system events, heartbeat state, delivery queues, fetch/proxy helpers,
  file helpers, approval types, and unrelated utilities.
- **`openclaw/plugin-sdk/config-runtime`** — a broad config compatibility barrel
  that still carries deprecated direct load/write helpers during the migration
  window.
- **`openclaw/extension-api`** — a bridge that gave plugins direct access to
  host-side helpers like the embedded agent runner.
- **`api.registerEmbeddedExtensionFactory(...)`** — a removed Pi-only bundled
  extension hook that could observe embedded-runner events such as
  `tool_result`.

The broad import surfaces are now **deprecated**. They still work at runtime,
but new plugins must not use them, and existing plugins should migrate before
the next major release removes them. The Pi-only embedded extension factory
registration API has been removed; use tool-result middleware instead.

OpenClaw does not remove or reinterpret documented plugin behavior in the same
change that introduces a replacement. Breaking contract changes must first go
through a compatibility adapter, diagnostics, docs, and a deprecation window.
That applies to SDK imports, manifest fields, setup APIs, hooks, and runtime
registration behavior.

  The backwards-compatibility layer will be removed in a future major release.
  Plugins that still import from these surfaces will break when that happens.
  Pi-only embedded extension factory registrations already no longer load.

## Why this changed

The old approach caused problems:

- **Slow startup** — importing one helper loaded dozens of unrelated modules
- **Circular dependencies** — broad re-exports made it easy to create import cycles
- **Unclear API surface** — no way to tell which exports were stable vs internal

The modern plugin SDK fixes this: each import path (`openclaw/plugin-sdk/\<subpath\>`)
is a small, self-contained module with a clear purpose and documented contract.

Legacy provider convenience seams for bundled channels are also gone.
Channel-branded helper seams were private mono-repo shortcuts, not stable
plugin contracts. Use narrow generic SDK subpaths instead. Inside the bundled
plugin workspace, keep provider-owned helpers in that plugin's own `api.ts` or
`runtime-api.ts`.

Current bundled provider examples:

- Anthropic keeps Claude-specific stream helpers in its own `api.ts` /
  `contract-api.ts` seam
- OpenAI keeps provider builders, default-model helpers, and realtime provider
  builders in its own `api.ts`
- OpenRouter keeps provider builder and onboarding/config helpers in its own
  `api.ts`

## Compatibility policy

For external plugins, compatibility work follows this order:

1. add the new contract
2. keep the old behavior wired through a compatibility adapter
3. emit a diagnostic or warning that names the old path and replacement
4. cover both paths in tests
5. document the deprecation and migration path
6. remove only after the announced migration window, usually in a major release

Maintainers can audit the current migration queue with
`pnpm plugins:boundary-report`. Use `pnpm plugins:boundary-report:summary` for
compact counts, `--owner <id>` for one plugin or compatibility owner, and
`pnpm plugins:boundary-report:ci` when a CI gate should fail on due
compatibility records, cross-owner reserved SDK imports, or unused reserved SDK
subpaths. The report groups deprecated
compatibility records by removal date, counts local code/docs references,
surfaces cross-owner reserved SDK imports, and summarizes the private
memory-host SDK bridge so compatibility cleanup stays explicit instead of
relying on ad hoc searches. Reserved SDK subpaths must have tracked owner usage;
unused reserved helper exports should be removed from the public SDK.

If a manifest field is still accepted, plugin authors can keep using it until
the docs and diagnostics say otherwise. New code should prefer the documented
replacement, but existing plugins should not break during ordinary minor
releases.

## How to migrate

    Bundled plugins should stop calling
    `api.runtime.config.loadConfig()` and
    `api.runtime.config.writeConfigFile(...)` directly. Prefer config that was
    already passed into the active call path. Long-lived handlers that need the
    current process snapshot can use `api.runtime.config.current()`. Long-lived
    agent tools should use the tool context's `ctx.getRuntimeConfig()` inside
    `execute` so a tool created before a config write still sees the refreshed
    runtime config.

    Config writes must go through the transactional helpers and choose an
    after-write policy:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    Use `afterWrite: { mode: "restart", reason: "..." }` when the caller knows
    the change requires a clean gateway restart, and
    `afterWrite: { mode: "none", reason: "..." }` only when the caller owns the
    follow-up and deliberately wants to suppress the reload planner.
    Mutation results include a typed `followUp` summary for tests and logging;
    the gateway remains responsible for applying or scheduling the restart.
    `loadConfig` and `writeConfigFile` remain as deprecated compatibility
    helpers for external plugins during the migration window and warn once with
    the `runtime-config-load-write` compatibility code. Bundled plugins and repo
    runtime code are protected by scanner guardrails in
    `pnpm check:deprecated-internal-config-api` and
    `pnpm check:no-runtime-action-load-config`: new production plugin usage
    fails outright, direct config writes fail, gateway server methods must use
    the request runtime snapshot, runtime channel send/action/client helpers
    must receive config from their boundary, and long-lived runtime modules have
    zero allowed ambient `loadConfig()` calls.

    New plugin code should also avoid importing the broad
    `openclaw/plugin-sdk/config-runtime` compatibility barrel. Use the narrow
    SDK subpath that matches the job:

    | Need | Import |
    | --- | --- |
    | Config types such as `OpenClawConfig` | `openclaw/plugin-sdk/config-types` |
    | Already-loaded config assertions and plugin-entry config lookup | `openclaw/plugin-sdk/plugin-config-runtime` |
    | Current runtime snapshot reads | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | Config writes | `openclaw/plugin-sdk/config-mutation` |
    | Session store helpers | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown table config | `openclaw/plugin-sdk/markdown-table-runtime` |
    | Group policy runtime helpers | `openclaw/plugin-sdk/runtime-group-policy` |
    | Secret input resolution | `openclaw/plugin-sdk/secret-input-runtime` |
    | Model/session overrides | `openclaw/plugin-sdk/model-session-runtime` |

    Bundled plugins and their tests are scanner-guarded against the broad
    barrel so imports and mocks stay local to the behavior they need. The broad
    barrel still exists for external compatibility, but new code should not
    depend on it.

    Bundled plugins must replace Pi-only
    `api.registerEmbeddedExtensionFactory(...)` tool-result handlers with
    runtime-neutral middleware.

    ```typescript
    // Pi and Codex runtime dynamic tools
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["pi", "codex"],
    });
    ```

    Update the plugin manifest at the same time:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["pi", "codex"]
      }
    }
    ```

    External plugins cannot register tool-result middleware because it can
    rewrite high-trust tool output before the model sees it.

    Approval-capable channel plugins now expose native approval behavior through
    `approvalCapability.nativeRuntime` plus the shared runtime-context registry.

    Key changes:

    - Replace `approvalCapability.handler.loadRuntime(...)` with
      `approvalCapability.nativeRuntime`
    - Move approval-specific auth/delivery off legacy `plugin.auth` /
      `plugin.approvals` wiring and onto `approvalCapability`
    - `ChannelPlugin.approvals` has been removed from the public channel-plugin
      contract; move delivery/native/render fields

---
domain: plugins
topic: "Plugin SDK Migration: Active Deprecations, Removal Timeline, and Warning Suppression"
type: reference
keywords:
  - active deprecations
  - removal timeline
  - warning suppression
  - SDK breaking changes
related:
  - plugins/sdk-migration
source: plugins/sdk-migration.md
---

Plugin SDK migration continued: specific API changes and suppressing deprecation warnings.

onto `approvalCapability`
    - `plugin.auth` remains for channel login/logout flows only; approval auth
      hooks there are no longer read by core
    - Register channel-owned runtime objects such as clients, tokens, or Bolt
      apps through `openclaw/plugin-sdk/channel-runtime-context`
    - Do not send plugin-owned reroute notices from native approval handlers;
      core now owns routed-elsewhere notices from actual delivery results
    - When passing `channelRuntime` into `createChannelManager(...)`, provide a
      real `createPluginRuntime().channel` surface. Partial stubs are rejected.

    See `/plugins/sdk-channel-plugins` for the current approval capability
    layout.

    If your plugin uses `openclaw/plugin-sdk/windows-spawn`, unresolved Windows
    `.cmd`/`.bat` wrappers now fail closed unless you explicitly pass
    `allowShellFallback: true`.

    ```typescript
    // Before
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // After
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Only set this for trusted compatibility callers that intentionally
      // accept shell-mediated fallback.
      allowShellFallback: true,
    });
    ```

    If your caller does not intentionally rely on shell fallback, do not set
    `allowShellFallback` and handle the thrown error instead.

    Search your plugin for imports from either deprecated surface:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

    Each export from the old surface maps to a specific modern import path:

    ```typescript
    // Before (deprecated backwards-compatibility layer)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // After (modern focused imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    For host-side helpers, use the injected plugin runtime instead of importing
    directly:

    ```typescript
    // Before (deprecated extension-api bridge)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // After (injected runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    The same pattern applies to other legacy bridge helpers:

    | Old import | Modern equivalent |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | session store helpers | `api.runtime.agent.session.*` |

    `openclaw/plugin-sdk/infra-runtime` still exists for external
    compatibility, but new code should import the focused helper surface it
    actually needs:

    | Need | Import |
    | --- | --- |
    | System event queue helpers | `openclaw/plugin-sdk/system-event-runtime` |
    | Heartbeat event and visibility helpers | `openclaw/plugin-sdk/heartbeat-runtime` |
    | Pending delivery queue drain | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | Channel activity telemetry | `openclaw/plugin-sdk/channel-activity-runtime` |
    | In-memory dedupe caches | `openclaw/plugin-sdk/dedupe-runtime` |
    | Safe local-file/media path helpers | `openclaw/plugin-sdk/file-access-runtime` |
    | Dispatcher-aware fetch | `openclaw/plugin-sdk/runtime-fetch` |
    | Proxy and guarded fetch helpers | `openclaw/plugin-sdk/fetch-runtime` |
    | SSRF dispatcher policy types | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | Approval request/resolution types | `openclaw/plugin-sdk/approval-runtime` |
    | Approval reply payload and command helpers | `openclaw/plugin-sdk/approval-reply-runtime` |
    | Error formatting helpers | `openclaw/plugin-sdk/error-runtime` |
    | Transport readiness waits | `openclaw/plugin-sdk/transport-ready-runtime` |
    | Secure token helpers | `openclaw/plugin-sdk/secure-random-runtime` |
    | Bounded async task concurrency | `openclaw/plugin-sdk/concurrency-runtime` |
    | Numeric coercion | `openclaw/plugin-sdk/number-runtime` |
    | Process-local async lock | `openclaw/plugin-sdk/async-lock-runtime` |
    | File locks | `openclaw/plugin-sdk/file-lock` |

    Bundled plugins are scanner-guarded against `infra-runtime`, so repo code
    cannot regress to the broad barrel.

    New channel route code should use `openclaw/plugin-sdk/channel-route`.
    The older route-key and comparable-target names remain as compatibility
    aliases during the migration window, but new plugins should use the route
    names that describe the behavior directly:

    | Old helper | Modern helper |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |
    | `ComparableChannelTarget` | `ChannelRouteParsedTarget` |
    | `resolveComparableTargetForChannel(...)` | `resolveRouteTargetForChannel(...)` |
    | `resolveComparableTargetForLoadedChannel(...)` | `resolveRouteTargetForLoadedChannel(...)` |
    | `comparableChannelTargetsMatch(...)` | `channelRouteTargetsMatchExact(...)` |
    | `comparableChannelTargetsShareRoute(...)` | `channelRouteTargetsShareConversation(...)` |

    The modern route helpers normalize `{ channel, to, accountId, threadId }`
    consistently across native approvals, reply suppression, inbound dedupe,
    cron delivery, and session routing. If your plugin owns custom target
    grammar, use `resolveChannelRouteTargetWithParser(...)` to adapt that
    parser into the same route target contract.

    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```

## Import path reference

  | Import path | Purpose | Key exports |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Canonical plugin entry helper | `definePluginEntry` |
  | `plugin-sdk/core` | Legacy umbrella re-export for channel entry definitions/builders | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Root config schema export | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Single-provider entry helper | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Focused channel entry definitions and builders | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChann

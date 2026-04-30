---
domain: plugins
topic: "SDK Import Subpaths: Complete Map of @openclaw/sdk Imports and APIs"
type: reference
keywords:
  - SDK subpaths
  - SDK imports
  - @openclaw/sdk
  - import paths
  - SDK API map
  - SDK reference
related:
  - plugins/sdk-overview
  - plugins/sdk-migration
source: plugins/sdk-subpaths.md
---

OpenClaw SDK import subpaths: the complete map of SDK package imports and which APIs each subpath provides.

The plugin SDK is exposed as a set of narrow subpaths under `openclaw/plugin-sdk/`.
This page catalogs the commonly used subpaths grouped by purpose. The generated
full list of 200+ subpaths lives in `scripts/lib/plugin-sdk-entrypoints.json`;
reserved bundled-plugin helper subpaths appear there but are implementation
detail unless a doc page explicitly promotes them. Maintainers can audit active
reserved helper subpaths with `pnpm plugins:boundary-report:summary`; unused
reserved helper exports fail the CI report instead of staying in the public SDK
as dormant compatibility debt.

For the plugin authoring guide, see [Plugin SDK overview](/plugins/sdk-overview).

## Plugin entry

| Subpath                                   | Key exports                                                                                                                                                                  |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`                 | `definePluginEntry`                                                                                                                                                          |
| `plugin-sdk/core`                         | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`                                       |
| `plugin-sdk/config-schema`                | `OpenClawSchema`                                                                                                                                                             |
| `plugin-sdk/provider-entry`               | `defineSingleProviderPluginEntry`                                                                                                                                            |
| `plugin-sdk/testing`                      | Broad compatibility barrel for legacy plugin tests; prefer focused test subpaths for new extension tests                                                                     |
| `plugin-sdk/plugin-test-api`              | Minimal `OpenClawPluginApi` mock builder for direct plugin registration unit tests                                                                                           |
| `plugin-sdk/agent-runtime-test-contracts` | Native agent-runtime adapter contract fixtures for auth profiles, delivery suppression, fallback classification, tool hooks, prompt overlays, schemas, and transcript repair |
| `plugin-sdk/channel-test-helpers`         | Channel account lifecycle, directory, send-config, runtime mock, hook, bundled channel entry, envelope timestamp, pairing reply, and generic channel contract test helpers   |
| `plugin-sdk/channel-target-testing`       | Shared channel target-resolution error-case test suite                                                                                                                       |
| `plugin-sdk/plugin-test-contracts`        | Plugin registration, package manifest, public artifact, runtime API, import side-effect, and direct import contract helpers                                                  |
| `plugin-sdk/plugin-test-runtime`          | Plugin runtime, registry, provider-registration, setup-wizard, and runtime task-flow fixtures for tests                                                                      |
| `plugin-sdk/provider-test-contracts`      | Provider runtime, auth, discovery, onboard, catalog, media capability, replay policy, realtime STT live-audio, web-search/fetch, and wizard contract helpers                 |
| `plugin-sdk/provider-http-test-mocks`     | Opt-in Vitest HTTP/auth mocks for provider tests that exercise `plugin-sdk/provider-http`                                                                                    |
| `plugin-sdk/test-env`                     | Test environment, fetch/network, disposable HTTP server, incoming request, live-test, temporary filesystem, and time-control fixtures                                        |
| `plugin-sdk/test-fixtures`                | Generic CLI, sandbox, skill, agent-message, system-event, module reload, bundled plugin path, terminal, chunking, auth-token, and typed-case test fixtures                   |
| `plugin-sdk/test-node-mocks`              | Focused Node builtin mock helpers for use inside Vitest `vi.mock("node:*")` factories                                                                                        |
| `plugin-sdk/migration`                    | Migration provider item helpers such as `createMigrationItem`, reason constants, item status markers, redaction helpers, and `summarizeMigrationItems`                       |
| `plugin-sdk/migration-runtime`            | Runtime migration helpers such as `copyMigrationFileItem`, `withCachedMigrationConfigRuntime`, and `writeMigrationReport`                                                    |

    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Root `openclaw.json` Zod schema export (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Shared setup wizard helpers, allowlist prompts, setup status builders |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Multi-account config/action-gate helpers, default-account fallback helpers |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, account-id normalization helpers |
    | `plugin-sdk/account-resolution` | Account lookup + default-fallback helpers |
    | `plugin-sdk/account-helpers` | Narrow account-list/account-action helpers |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline`, `resolveChannelSourceReplyDeliveryMode` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`, `resolveChannelDmAccess`, `resolveChannelDmAllowFrom`, `resolveChannelDmPolicy`, `normalizeChannelDmPolicy`, `normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | Shared channel config schema primitives and generic builder |
    | `plugin-sdk/bundled-channel-config-schema` | Bundled OpenClaw channel config schemas for maintained bundled plugins only |
    | `plugin-sdk/channel-config-schema-legacy` | Deprecated compatibility alias for bundled-channel config schemas |
    | `plugin-sdk/telegram-command-config` | Telegram custom-command normalization/validation helpers with bundled-contract fallback |
    | `plugin-sdk/command-gating` | Narrow command authorization gate helpers |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink`, `createChannelRunQueue`, draft stream lifecycle/finalization helpers |
    | `plugin-sdk/inbound-envelope` | Shared inbound route + envelope builder helpers |
    | `plugin-sdk/inbound-reply-dispatch` | Shared inbound record-and-dispatch helpers |
    | `plugin-sdk/messaging-targets` | Target parsing/matching helpers |
    | `plugin-sdk/outbound-media` | Shared outbound media loading helpers |
    | `plugin-sdk/outbound-send-deps` | Lightweight outbound send dependency lookup for channel adapters |
    | `plugin-sdk/outbound-runtime` | Outbound delivery, identity, send delegate, session, formatting, and payload planning helpers |
    | `plugin-sdk/poll-runtime` | Narrow poll normalization helpers |
    | `plugin-sdk/thread-bindings-runtime` | Thread-binding lifecycle and adapter helpers |
    | `plugin-sdk/agent-media-payload` | Legacy agent media payload builder |
    | `plugin-sdk/conversation-runtime` | Conversation/thread binding, pairing, and configured-binding helpers |
    | `plugin-sdk/runtime-config-snapshot` | Runtime config snapshot helper |
    | `plugin-sdk/runtime-group-policy` | Runtime group-policy resolution helpers |
    | `plugin-sdk/channel-status` | Shared channel status snapshot/summ

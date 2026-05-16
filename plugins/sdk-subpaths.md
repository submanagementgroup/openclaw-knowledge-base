---
domain: plugins
topic: "SDK Subpaths: Plugin-Scoped Config and State"
type: reference
keywords:
  - subpaths
  - plugin subpaths
  - config subpath
  - state subpath
  - plugin config path
source: plugins/sdk-subpaths.md
---

The plugin SDK is exposed as a set of narrow public subpaths under
`openclaw/plugin-sdk/`. This page catalogs the commonly used subpaths grouped by
purpose. The generated compiler entrypoint inventory lives in
`scripts/lib/plugin-sdk-entrypoints.json`; package exports are the public subset
after subtracting repo-local test/internal subpaths listed in
`scripts/lib/plugin-sdk-private-local-only-subpaths.json`. Maintainers can audit
the public export count with `pnpm plugin-sdk:surface` and active reserved
helper subpaths with `pnpm plugins:boundary-report:summary`; unused reserved
helper exports fail the CI report instead of staying in the public SDK as
dormant compatibility debt.

For the plugin authoring guide, see [Plugin SDK overview](/plugins/sdk-overview).

## Plugin entry

| Subpath                        | Key exports                                                                                                                                                            |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                    |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema` |
| `plugin-sdk/config-schema`     | `OpenClawSchema`                                                                                                                                                       |
| `plugin-sdk/provider-entry`    | `defineSingleProviderPluginEntry`                                                                                                                                      |
| `plugin-sdk/migration`         | Migration provider item helpers such as `createMigrationItem`, reason constants, item status markers, redaction helpers, and `summarizeMigrationItems`                 |
| `plugin-sdk/migration-runtime` | Runtime migration helpers such as `copyMigrationFileItem`, `withCachedMigrationConfigRuntime`, and `writeMigrationReport`                                              |

### Deprecated compatibility and test helpers

These subpaths remain package exports for older plugins and OpenClaw test suites,
but new code should not add imports from them: `agent-runtime-test-contracts`,
`channel-contract-testing`, `channel-target-testing`, `channel-test-helpers`,
`plugin-test-api`, `plugin-test-contracts`, `provider-http-test-mocks`,
`provider-test-contracts`, `test-env`, `test-fixtures`, `test-node-mocks`,
`testing`, `channel-runtime`, `compat`, `config-types`, `infra-runtime`,
`text-runtime`, and `zod`. Import `zod` directly from `zod` in new plugin code.
`plugin-test-runtime` is still an active focused test helper subpath.

### Reserved bundled plugin helper subpaths

These subpaths are plugin-owned compatibility surfaces reserved for their owning
bundled plugin, not general SDK APIs: `plugin-sdk/codex-mcp-projection` and
`plugin-sdk/codex-native-task-runtime`. Cross-owner extension imports are blocked
by package contract guardrails.

### Deprecated unused public subpaths

These public subpaths existed for at least one month and currently have no
bundled extension production imports. They remain importable for compatibility,
but new plugin code should use focused, actively consumed SDK subpaths instead:
`agent-config-primitives`, `channel-config-schema-legacy`,
`channel-reply-pipeline`, `channel-runtime`, `channel-secret-runtime`,
`command-auth`, `compat`, `config-runtime`, `config-schema`, `discord`,
`group-access`, `infra-runtime`, `matrix`, `mattermost`,
`media-generation-runtime-shared`, `memory-core-engine-runtime`,
`memory-core-host-multimodal`, `memory-core-host-query`,
`music-generation-core`, `self-hosted-provider-setup`, `telegram-account`,
`telegram-command-config`, and `zalouser`.

### Deprecated rare public subpaths

Public subpaths currently used by only one or two bundled plugin owners are also
deprecated for new plugin code. They remain package exports for compatibility,
but new code should prefer actively shared SDK seams or plugin-owned package
APIs. Maintainers track the exact set in
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json` and the current budget
with `pnpm plugin-sdk:surface`.

### Deprecated broad barrels

These broad re-export barrels remain buildable for OpenClaw source and
compatibility checks, but new code should prefer focused SDK subpaths:
`agent-runtime`, `channel-lifecycle`, `channel-runtime`, `cli-runtime`,
`compat`, `config-types`, `conversation-runtime`, `hook-runtime`,
`infra-runtime`, `media-runtime`, `plugin-runtime`, `security-runtime`, and
`text-runtime`. `channel-runtime`, `compat`, `config-types`, `infra-runtime`,
and `text-runtime` remain package exports only for backwards compatibility; use
focused channel/runtime subpaths, `config-contracts`, `string-coerce-runtime`,
`text-chunking`, `text-utility-runtime`, and `logging-core` instead.

### Channel subpaths

| Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Root `openclaw.json` Zod schema export (`OpenClawSchema`) |
    | `plugin-sdk/json-schema-runtime` | Cached JSON Schema validation helper for plugin-owned schemas |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Shared setup wizard helpers, allowlist prompts, setup status builders |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | Deprecated compatibility alias; use `plugin-sdk/setup-runtime` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Multi-account config/action-gate helpers, default-account fallback helpers |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, account-id normalization helpers |
    | `plugin-sdk/account-resolution` | Account lookup + default-fallback helpers |
    | `plugin-sdk/account-helpers` | Narrow account-list/account-action helpers |
    | `plugin-sdk/access-groups` | Access-group allowlist parsing and redacted group diagnostics helpers |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | Legacy reply pipeline helpers. New channel reply pipeline code should use `createChannelMessageReplyPipeline` and `resolveChannelMessageSourceReplyDeliveryMode` from `plugin-sdk/channel-message`. |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`, `resolveChannelDmAccess`, `resolveChannelDmAllowFrom`, `resolveChannelDmPolicy`, `normalizeChannelDmPolicy`, `normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | Shared channel config schema primitives plus Zod and direct JSON/TypeBox builders |
    | `plugin-sdk/bundled-channel-config-schema` | Bundled OpenClaw channel config schemas for maintained bundled plugins only |
    | `plugin-sdk/channel-config-schema-legacy` | Deprecated compatibility alias for bundled-channel config schemas |
    | `plugin-sdk/telegram-command-config` | Telegram custom-command normalization/validation helpers with bundled-contract fallback |
    | `plugin-sdk/command-gating` | Narrow command authorization gate helpers |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress` | Deprecated low-level channel ingress compatibility facade. New receive paths should use `plugin-sdk/channel-ingress-runtime`. |
    | `plugin-sdk/channel-ingress-runtime` | Experimental high-level channel ingress runtime resolver and route fact builders for migrated channel receive paths. Prefer this over assembling effective allowlists, command allowlists, and legacy projections in each plugin. See [Channel ingress API](/plugins/sdk-channel-ingress). |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink`, `createChannelRunQueue`, and legacy draft stream lifecycle helpers. New preview finalization code should use `plugin-sdk/channel-message`. |
    | `plugin-sdk/channel-message` | Cheap message lifecycle contract helpers such as `defineChannelMessageAdapter`, `createChannelMessageAdapterFromOutbound`, `createChannelMessageReplyPipeline`, `createReplyPrefixContext`, `resolveChannelMessageSourceReplyDeliveryMode`, durable-final capability derivation, capability proof helpers for send/receipt/side-effect capabilities, `MessageReceiveContext`, receive ack policy proofs, `defineFinalizableLivePreviewAdapter`, `deliverWithFinalizableLivePreviewAdapter`, live-preview and live-finalizer capability proofs, durable recovery state, `RenderedMessageBatch`, message receipt types, and receipt id helpers. See [Channel message API](/plugins/sdk-channel-message). Legacy reply-dispatch facades are deprecated compatibility only. |
    | `plugin-sdk/channel-message-runtime` | Runtime delivery helpers that may load outbound delivery, including `deliverInboundReplyWithMessageSendContext`, `sendDurableMessageBatch`, and `withDurableMessageSendContext`. Deprecated reply-dispatch bridges remain importable for compatibility dispatchers only. Use from monitor/send runtime modules, not hot plugin bootstrap files. |
    | `plugin-sdk/inbound-envelope` | Shared inbound route + envelope builder helpers |
    | `plugin-sdk/inbound-reply-dispatch` | Legacy shared inbound record-and-dispatch helpers, visible/final dispatch predicates, and deprecated `deliverDurableInboundReplyPayload` compatibility for prepared channel dispatchers. New channel receive/dispatch code should import runtime lifecycle helpers from `plugin-sdk/channel-message-runtime`. |
    | `plugin-sdk/messaging-targets` | Target parsing/matching helpers |
    | `plugin-sdk/outbound-media` | Shared outbound media loading helpers |
    | `plugin-sdk/outbound-send-deps` | Lightweight outbound send dependency lookup for channel adapters |
    | `plugin-sdk/outbound-runtime` | Outbound identity, send delegate, session, formatting, and payload planning helpers. Direct delivery helpers such as `deliverOutboundPayloads` are deprecated compatibility substrate; use `plugin-sdk/channel-message-runtime` for new send paths. |
    | `plugin-sdk/poll-runtime` | Narrow poll normalization helpers |
    | `plugin-sdk/thread-bindings-runtime` | Thread-binding lifecycle and adapter helpers |
    | `plugin-sdk/agent-media-payload` | Legacy agent media payload builder |
    | `plugin-sdk/conversation-runtime` | Conversation/thread binding, pairing, and configured-binding helpers |
    | `plugin-sdk/runtime-config-snapshot` | Runtime config snapshot helper |
    | `plugin-sdk/runtime-group-policy` | Runtime group-policy resolution helpers |
    | `plugin-sdk/channel-status` | Shared channel status snapshot/summary helpers |
    | `plugin-sdk/channel-config-primitives` | Narrow channel config-schema primitives |
    | `plugin-sdk/channel-config-writes` | Channel config-write authorization helpers |
    | `plugin-sdk/channel-plugin-common` | Shared channel plugin prelude exports |
    | `plugin-sdk/allowlist-config-edit` | Allowlist config edit/read helpers |
    | `plugin-sdk/group-access` | Shared group-access decision helpers |
    | `plugin-sdk/direct-dm` | Shared direct-DM auth/guard helpers |
    | `plugin-sdk/discord` | Deprecated Discord compatibility facade for published `@openclaw/discord@2026.3.13` and tracked owner compatibility; new plugins should use generic channel SDK subpaths |
    | `plugin-sdk/telegram-account` | Deprecated Telegram account-resolution compatibility facade for tracked owner compatibility; new plugins should use injected runtime helpers or generic channel SDK subpaths |
    | `plugin-sdk/zalouser` | Deprecated Zalo Personal compatibility facade for published Lark/Zalo packages that still import sender command authorization; new plugins should use `plugin-sdk/command-auth` |
    | `plugin-sdk/interactive-runtime` | Semantic message presentation, delivery, and legacy interactive reply helpers. See [Message Presentation](/plugins/message-presentation) |
    | `plugin-sdk/channel-inbound` | Compatibility barrel for inbound debounce, mention matching, mention-policy helpers, and envelope helpers |
    | `plugin-sdk/channel-inbound-debounce` | Narrow inbound debounce helpers |
    | `plugin-sdk/channel-mention-gating` | Narrow mention-policy, mention marker, and mention text helpers without the broader inbound runtime surface |
    | `plugin-sdk/channel-envelope` | Narrow inbound envelope formatting helpers |
    | `plugin-sdk/channel-location` | Channel location context and formatting helpers |
    | `plugin-sdk/channel-logging` | Channel logging helpers for inbound drops and typing/ack failures |
    | `plugin-sdk/channel-send-result` | Reply result types |
    | `plugin-sdk/channel-actions` | Channel message-action helpers, plus deprecated native schema helpers kept for plugin compatibility |
    | `plugin-sdk/channel-route` | Shared route normalization, parser-driven target resolution, thread-id stringification,
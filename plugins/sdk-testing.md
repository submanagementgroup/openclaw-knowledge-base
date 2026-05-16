---
domain: plugins
topic: "Plugin SDK Testing"
type: procedure
keywords:
  - plugin testing
  - SDK testing
  - test plugin
  - unit test
  - integration test
source: plugins/sdk-testing.md
---

Reference for test utilities, patterns, and lint enforcement for OpenClaw
plugins.

> **Note:** **Looking for test examples?** The how-to guides include worked test examples:
  [Channel plugin tests](/plugins/sdk-channel-plugins#step-6-test) and
  [Provider plugin tests](/plugins/sdk-provider-plugins#step-6-test).


## Test utilities

These test-helper subpaths are repo-local source entrypoints for OpenClaw's own
bundled plugin tests. They are not package exports for third-party plugins.

**Plugin API mock import:** `openclaw/plugin-sdk/plugin-test-api`

**Agent runtime contract import:** `openclaw/plugin-sdk/agent-runtime-test-contracts`

**Channel contract import:** `openclaw/plugin-sdk/channel-contract-testing`

**Channel test helper import:** `openclaw/plugin-sdk/channel-test-helpers`

**Channel target test import:** `openclaw/plugin-sdk/channel-target-testing`

**Plugin contract import:** `openclaw/plugin-sdk/plugin-test-contracts`

**Plugin runtime test import:** `openclaw/plugin-sdk/plugin-test-runtime`

**Provider contract import:** `openclaw/plugin-sdk/provider-test-contracts`

**Provider HTTP mock import:** `openclaw/plugin-sdk/provider-http-test-mocks`

**Environment/network test import:** `openclaw/plugin-sdk/test-env`

**Generic fixture import:** `openclaw/plugin-sdk/test-fixtures`

**Node builtin mock import:** `openclaw/plugin-sdk/test-node-mocks`

Prefer the focused subpaths below for new plugin tests. The broad
`openclaw/plugin-sdk/testing` barrel is legacy compatibility only.
Repo guardrails reject new real imports from `plugin-sdk/testing` and
`plugin-sdk/test-utils`; those names remain only as deprecated compatibility
surfaces for compatibility-record tests.

```typescript
import {
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/channel-feedback";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";
import { AUTH_PROFILE_RUNTIME_CONTRACT } from "openclaw/plugin-sdk/agent-runtime-test-contracts";
import { createTestPluginApi } from "openclaw/plugin-sdk/plugin-test-api";
import { expectChannelInboundContextContract } from "openclaw/plugin-sdk/channel-contract-testing";
import { createStartAccountContext } from "openclaw/plugin-sdk/channel-test-helpers";
import { describePluginRegistrationContract } from "openclaw/plugin-sdk/plugin-test-contracts";
import { registerSingleProviderPlugin } from "openclaw/plugin-sdk/plugin-test-runtime";
import { describeOpenAIProviderRuntimeContract } from "openclaw/plugin-sdk/provider-test-contracts";
import { getProviderHttpMocks } from "openclaw/plugin-sdk/provider-http-test-mocks";
import { withEnv, withFetchPreconnect, withServer } from "openclaw/plugin-sdk/test-env";
import {
  bundledPluginRoot,
  createCliRuntimeCapture,
  typedCases,
} from "openclaw/plugin-sdk/test-fixtures";
import { mockNodeBuiltinModule } from "openclaw/plugin-sdk/test-node-mocks";
```

### Available exports

| Export                                               | Purpose                                                                                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `createTestPluginApi`                                | Build a minimal plugin API mock for direct registration unit tests. Import from `plugin-sdk/plugin-test-api`                             |
| `AUTH_PROFILE_RUNTIME_CONTRACT`                      | Shared auth-profile contract fixture for native agent runtime adapters. Import from `plugin-sdk/agent-runtime-test-contracts`            |
| `DELIVERY_NO_REPLY_RUNTIME_CONTRACT`                 | Shared delivery suppression contract fixture for native agent runtime adapters. Import from `plugin-sdk/agent-runtime-test-contracts`    |
| `OUTCOME_FALLBACK_RUNTIME_CONTRACT`                  | Shared fallback-classification contract fixture for native agent runtime adapters. Import from `plugin-sdk/agent-runtime-test-contracts` |
| `createParameterFreeTool`                            | Build dynamic-tool schema fixtures for native runtime contract tests. Import from `plugin-sdk/agent-runtime-test-contracts`              |
| `expectChannelInboundContextContract`                | Assert channel inbound context shape. Import from `plugin-sdk/channel-contract-testing`                                                  |
| `installChannelOutboundPayloadContractSuite`         | Install channel outbound payload contract cases. Import from `plugin-sdk/channel-contract-testing`                                       |
| `createStartAccountContext`                          | Build channel account lifecycle contexts. Import from `plugin-sdk/channel-test-helpers`                                                  |
| `installChannelActionsContractSuite`                 | Install generic channel message-action contract cases. Import from `plugin-sdk/channel-test-helpers`                                     |
| `installChannelSetupContractSuite`                   | Install generic channel setup contract cases. Import from `plugin-sdk/channel-test-helpers`                                              |
| `installChannelStatusContractSuite`                  | Install generic channel status contract cases. Import from `plugin-sdk/channel-test-helpers`                                             |
| `expectDirectoryIds`                                 | Assert channel directory ids from a directory-list function. Import from `plugin-sdk/channel-test-helpers`                               |
| `assertBundledChannelEntries`                        | Assert bundled channel entrypoints expose the expected public contract. Import from `plugin-sdk/channel-test-helpers`                    |
| `formatEnvelopeTimestamp`                            | Format deterministic envelope timestamps. Import from `plugin-sdk/channel-test-helpers`                                                  |
| `expectPairingReplyText`                             | Assert channel pairing reply text and extract its code. Import from `plugin-sdk/channel-test-helpers`                                    |
| `describePluginRegistrationContract`                 | Install plugin registration contract checks. Import from `plugin-sdk/plugin-test-contracts`                                              |
| `registerSingleProviderPlugin`                       | Register one provider plugin in loader smoke tests. Import from `plugin-sdk/plugin-test-runtime`                                         |
| `registerProviderPlugin`                             | Capture all provider kinds from one plugin. Import from `plugin-sdk/plugin-test-runtime`                                                 |
| `registerProviderPlugins`                            | Capture provider registrations across multiple plugins. Import from `plugin-sdk/plugin-test-runtime`                                     |
| `requireRegisteredProvider`                          | Assert that a provider collection contains an id. Import from `plugin-sdk/plugin-test-runtime`                                           |
| `createRuntimeEnv`                                   | Build a mocked CLI/plugin runtime environment. Import from `plugin-sdk/plugin-test-runtime`                                              |
| `createPluginSetupWizardStatus`                      | Build setup status helpers for channel plugins. Import from `plugin-sdk/plugin-test-runtime`                                             |
| `describeOpenAIProviderRuntimeContract`              | Install provider-family runtime contract checks. Import from `plugin-sdk/provider-test-contracts`                                        |
| `expectPassthroughReplayPolicy`                      | Assert provider replay policies pass through provider-owned tools and metadata. Import from `plugin-sdk/provider-test-contracts`         |
| `runRealtimeSttLiveTest`                             | Run a live realtime STT provider test with shared audio fixtures. Import from `plugin-sdk/provider-test-contracts`                       |
| `normalizeTranscriptForMatch`                        | Normalize live transcript output before fuzzy assertions. Import from `plugin-sdk/provider-test-contracts`                               |
| `expectExplicitVideoGenerationCapabilities`          | Assert video providers declare explicit generation mode capabilities. Import from `plugin-sdk/provider-test-contracts`                   |
| `expectExplicitMusicGenerationCapabilities`          | Assert music providers declare explicit generation/edit capabilities. Import from `plugin-sdk/provider-test-contracts`                   |
| `mockSuccessfulDashscopeVideoTask`                   | Install a successful DashScope-compatible video task response. Import from `plugin-sdk/provider-test-contracts`                          |
| `getProviderHttpMocks`                               | Access opt-in provider HTTP/auth Vitest mocks. Import from `plugin-sdk/provider-http-test-mocks`                                         |
| `installProviderHttpMockCleanup`                     | Reset provider HTTP/auth mocks after each test. Import from `plugin-sdk/provider-http-test-mocks`                                        |
| `installCommonResolveTargetErrorCases`               | Shared test cases for target resolution error handling. Import from `plugin-sdk/channel-target-testing`                                  |
| `shouldAckReaction`                                  | Check whether a channel should add an ack reaction. Import from `plugin-sdk/channel-feedback`                                            |
| `removeAckReactionAfterReply`                        | Remove ack reaction after reply delivery. Import from `plugin-sdk/channel-feedback`                                                      |
| `createTestRegistry`                                 | Build a channel plugin registry fixture. Import from `plugin-sdk/plugin-test-runtime` or `plugin-sdk/channel-test-helpers`               |
| `createEmptyPluginRegistry`                          | Build an empty plugin registry fixture. Import from `plugin-sdk/plugin-test-runtime` or `plugin-sdk/channel-test-helpers`                |
| `setActivePluginRegistry`                            | Install a registry fixture for plugin runtime tests. Import from `plugin-sdk/plugin-test-runtime` or `plugin-sdk/channel-test-helpers`   |
| `createRequestCaptureJsonFetch`                      | Capture JSON fetch requests in media helper tests. Import from `plugin-sdk/test-env`                                                     |
| `withServer`                                         | Run tests against a disposable local HTTP server. Import from `plugin-sdk/test-env`                                                      |
| `createMockIncomingRequest`                          | Build a minimal incoming HTTP request object. Import from `plugin-sdk/test-env`                                                          |
| `withFetchPreconnect`                                | Run fetch tests with preconnect hooks installed. Import from `plugin-sdk/test-env`                                                       |
| `withEnv` / `withEnvAsync`                           | Temporarily patch environment variables. Import from `plugin-sdk/test-env`                                                               |
| `createTempHomeEnv` / `withTempHome` / `withTempDir` | Create isolated filesystem test fixtures. Import from `plugin-sdk/test-env`                                                              |
| `createMockServerResponse`                           | Create a minimal HTTP server response mock. Import from `plugin-sdk/test-env`                                                            |
| `createCliRunti
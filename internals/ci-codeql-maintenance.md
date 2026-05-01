---
domain: internals
topic: "CI CodeQL, Maintenance Workflows, Local Gates, and Testbox"
type: reference
keywords:
  - CodeQL
  - security categories
  - quality categories
  - Maintenance workflows
  - Docs Agent
  - Test Performance Agent
  - local check gates
  - Testbox
related:
  - internals/ci-pipeline-overview
  - internals/ci-release-workflows
  - internals/ci-package-acceptance
  - internals/ci-codeql-maintenance
source: ci.md
---

## CodeQL

The `CodeQL` workflow is intentionally a narrow first-pass security scanner, not the full repository sweep. Daily, manual, and non-draft pull request guard runs scan Actions workflow code plus the highest-risk JavaScript/TypeScript surfaces with high-confidence security queries filtered to high/critical `security-severity`.

The pull request guard stays light: it only starts for changes under `.github/actions`, `.github/codeql`, `.github/workflows`, `packages`, or `src`, and it runs the same high-confidence security matrix as the scheduled workflow. Android and macOS CodeQL stay out of PR defaults.

### Security categories

| Category                                          | Surface                                                                                                                                |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | Auth, secrets, sandbox, cron, and gateway baseline                                                                                     |
| `/codeql-security-high/channel-runtime-boundary`  | Core channel implementation contracts plus the channel plugin runtime, gateway, Plugin SDK, secrets, audit touchpoints                 |
| `/codeql-security-high/network-ssrf-boundary`     | Core SSRF, IP parsing, network guard, web-fetch, and Plugin SDK SSRF policy surfaces                                                   |
| `/codeql-security-high/mcp-process-tool-boundary` | MCP servers, process execution helpers, outbound delivery, and agent tool-execution gates                                              |
| `/codeql-security-high/plugin-trust-boundary`     | Plugin install, loader, manifest, registry, runtime-dependency staging, source-loading, and Plugin SDK package contract trust surfaces |

### Platform-specific security shards

- `CodeQL Android Critical Security` — scheduled Android security shard. Builds the Android app manually for CodeQL on the smallest Blacksmith Linux runner accepted by workflow sanity. Uploads under `/codeql-critical-security/android`.
- `CodeQL macOS Critical Security` — weekly/manual macOS security shard. Builds the macOS app manually for CodeQL on Blacksmith macOS, filters dependency build results out of uploaded SARIF, and uploads under `/codeql-critical-security/macos`. Kept outside daily defaults because macOS build dominates runtime even when clean.

### Critical Quality categories

`CodeQL Critical Quality` is the matching non-security shard. It runs only error-severity, non-security JavaScript/TypeScript quality queries over narrow high-value surfaces on the smaller Blacksmith Linux runner. Its pull request guard is intentionally smaller than the scheduled profile: non-draft PRs only run the matching `agent-runtime-boundary`, `config-boundary`, `core-auth-secrets`, `channel-runtime-boundary`, `gateway-runtime-boundary`, `memory-runtime-boundary`, `mcp-process-runtime-boundary`, `provider-runtime-boundary`, `session-diagnostics-boundary`, `plugin-boundary`, `plugin-sdk-package-contract`, and `plugin-sdk-reply-runtime` shards for agent command/model/tool execution and reply dispatch code, config schema/migration/IO code, auth/secrets/sandbox/security code, core channel and bundled channel plugin runtime, gateway protocol/server-method, memory runtime/SDK glue, MCP/process/outbound delivery, provider runtime/model catalog, session diagnostics/delivery queues, plugin loader, Plugin SDK/package-contract, or Plugin SDK reply runtime changes. CodeQL config and quality workflow changes run all twelve PR quality shards.

Manual dispatch accepts:

```
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

The narrow profiles are teaching/iteration hooks for running one quality shard in isolation.

| Category                                                | Surface                                                                                                                                                           |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | Auth, secrets, sandbox, cron, and gateway security boundary code                                                                                                  |
| `/codeql-critical-quality/config-boundary`              | Config schema, migration, normalization, and IO contracts                                                                                                         |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Gateway protocol schemas and server method contracts                                                                                                              |
| `/codeql-critical-quality/channel-runtime-boundary`     | Core channel and bundled channel plugin implementation contracts                                                                                                  |
| `/codeql-critical-quality/agent-runtime-boundary`       | Command execution, model/provider dispatch, auto-reply dispatch and queues, and ACP control-plane runtime contracts                                               |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | MCP servers and tool bridges, process supervision helpers, and outbound delivery contracts                                                                        |
| `/codeql-critical-quality/memory-runtime-boundary`      | Memory host SDK, memory runtime facades, memory Plugin SDK aliases, memory runtime activation glue, and memory doctor commands                                    |
| `/codeql-critical-quality/session-diagnostics-boundary` | Reply queue internals, session delivery queues, outbound session binding/delivery helpers, diagnostic event/log bundle surfaces, and session doctor CLI contracts |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | Plugin SDK inbound reply dispatch, reply payload/chunking/runtime helpers, channel reply options, delivery queues, and session/thread binding helpers             |
| `/codeql-critical-quality/provider-runtime-boundary`    | Model catalog normalization, provider auth and discovery, provider runtime registration, provider defaults/catalogs, and web/search/fetch/embedding registries    |
| `/codeql-critical-quality/ui-control-plane`             | Control UI bootstrap, local persistence, gateway control flows, and task control-plane runtime contracts                                                          |
| `/codeql-critical-quality/web-media-runtime-boundary`   | Core web fetch/search, media IO, media understanding, image-generation, and media-generation runtime contracts                                                    |
| `/codeql-critical-quality/plugin-boundary`              | Loader, registry, public-surface, and Plugin SDK entrypoint contracts                                                                                             |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | Published package-side Plugin SDK source and plugin package contract helpers                                                                                      |

Quality stays separate from security so quality findings can be scheduled, measured, disabled, or expanded without obscuring security signal. Swift, Python, and bundled-plugin CodeQL expansion should be added back as scoped or sharded follow-up work only after the narrow profiles have stable runtime and signal.


## Maintenance workflows

### Docs Agent

The `Docs Agent` workflow is an event-driven Codex maintenance lane for keeping existing docs aligned with recently landed changes. It has no pure schedule: a successful non-bot push CI run on `main` can trigger it, and manual dispatch can run it directly. Workflow-run invocations skip when `main` has moved on or when another non-skipped Docs Agent run was created in the last hour. When it runs, it reviews the commit range from the previous non-skipped Docs Agent source SHA to current `main`, so one hourly run can cover all main changes accumulated since the last docs pass.

### Test Performance Agent

The `Test Performance Agent` workflow is an event-driven Codex maintenance lane for slow tests. It has no pure schedule: a successful non-bot push CI run on `main` can trigger it, but it skips if another workflow-run invocation already ran or is running that UTC day. Manual dispatch bypasses that daily activity gate. The lane builds a full-suite grouped Vitest performance report, lets Codex make only small coverage-preserving test performance fixes instead of broad refactors, then reruns the full-suite report and rejects changes that reduce the passing baseline test count. If the baseline has failing tests, Codex may fix only obvious failures and the after-agent full-suite report must pass before anything is committed. When `main` advances before the bot push lands, the lane rebases the validated patch, reruns `pnpm check:changed`, and retries the push; conflicting stale patches are skipped. It uses GitHub-hosted Ubuntu so the Codex action can keep the same drop-sudo safety posture as the docs agent.

### Duplicate PRs After Merge

The `Duplicate PRs After Merge` workflow is a manual maintainer workflow for post-land duplicate cleanup. It defaults to dry-run and only closes explicitly listed PRs when `apply=true`. Before mutating GitHub, it verifies that the landed PR is merged and that each duplicate has either a shared referenced issue or overlapping changed hunks.

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```


## Local check gates and changed routing

Local changed-lane logic lives in `scripts/changed-lanes.mjs` and is executed by `scripts/check-changed.mjs`. That local check gate is stricter about architecture boundaries than the broad CI platform scope:

- core production changes run core prod and core test typecheck plus core lint/guards;
- core test-only changes run only core test typecheck plus core lint;
- extension production changes run extension prod and extension test typecheck plus extension lint;
- extension test-only changes run extension test typecheck plus extension lint;
- public Plugin SDK or plugin-contract changes expand to extension typecheck because extensions depend on those core contracts (Vitest extension sweeps stay explicit test work);
- release metadata-only version bumps run targeted version/config/root-dependency checks;
- unknown root/config changes fail safe to all check lanes.

Local changed-test routing lives in `scripts/test-projects.test-support.mjs` and is intentionally cheaper than `check:changed`: direct test edits run themselves, source edits prefer explicit mappings, then sibling tests and import-graph dependents. Shared group-room delivery config is one of the explicit mappings: changes to the group visible-reply config, source reply delivery mode, or the message-tool system prompt route through the core reply tests plus Discord and Slack delivery regressions so a shared default change fails before the first PR push. Use `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` only when the change is harness-wide enough that the cheap mapped set is not a trustworthy proxy.


## Testbox validation

Run Testbox from the repo root and prefer a fresh warmed box for broad proof. Before spending a slow gate on a box that was reused, expired, or just reported an unexpectedly large sync, run `pnpm testbox:sanity` inside the box first.

The sanity check fails fast when required root files such as `pnpm-lock.yaml` disappeared or when `git status --short` shows at least 200 tracked deletions. That usually means the remote sync state is not a trustworthy copy of the PR; stop that box and warm a fresh one instead of debugging the product test failure. For intentional large-deletion PRs, set `OPENCLAW_TESTBOX_ALLOW_MASS_DELETIONS=1` for that sanity run.

`pnpm testbox:run` also terminates a local Blacksmith CLI invocation that stays in the sync phase for more than five minutes without post-sync output. Set `OPENCLAW_TESTBOX_SYNC_TIMEOUT_MS=0` to disable that guard, or use a larger millisecond value for unusually large local diffs.


## Related

- [Install overview](/install)
- [Development channels](/install/development-channels)


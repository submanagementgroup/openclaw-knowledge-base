---
domain: internals
topic: "CI/CD Configuration"
type: reference
keywords:
  - CI
  - continuous integration
  - GitHub Actions
  - CI pipeline
source: ci.md
---

OpenClaw CI runs on every push to `main` and every pull request. The `preflight` job classifies the diff and turns expensive lanes off when only unrelated areas changed. Manual `workflow_dispatch` runs intentionally bypass smart scoping and fan out the full graph for release candidates and broad validation. Android lanes stay opt-in through `include_android`. Release-only plugin coverage lives in the separate [`Plugin Prerelease`](#plugin-prerelease) workflow and only runs from [`Full Release Validation`](#full-release-validation) or an explicit manual dispatch.

## Pipeline overview

| Job                              | Purpose                                                                                                   | When it runs                       |
| -------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| `preflight`                      | Detect docs-only changes, changed scopes, changed extensions, and build the CI manifest                   | Always on non-draft pushes and PRs |
| `security-scm-fast`              | Private key detection and workflow audit via `zizmor`                                                     | Always on non-draft pushes and PRs |
| `security-dependency-audit`      | Dependency-free production lockfile audit against npm advisories                                          | Always on non-draft pushes and PRs |
| `security-fast`                  | Required aggregate for the fast security jobs                                                             | Always on non-draft pushes and PRs |
| `check-dependencies`             | Production Knip dependency-only pass plus the unused-file allowlist guard                                 | Node-relevant changes              |
| `build-artifacts`                | Build `dist/`, Control UI, built-artifact checks, and reusable downstream artifacts                       | Node-relevant changes              |
| `checks-fast-core`               | Fast Linux correctness lanes such as bundled/plugin-contract/protocol checks                              | Node-relevant changes              |
| `checks-fast-contracts-channels` | Sharded channel contract checks with a stable aggregate check result                                      | Node-relevant changes              |
| `checks-node-core-test`          | Core Node test shards, excluding channel, bundled, contract, and extension lanes                          | Node-relevant changes              |
| `check`                          | Sharded main local gate equivalent: prod types, lint, guards, test types, and strict smoke                | Node-relevant changes              |
| `check-additional`               | Architecture, sharded boundary/prompt drift, extension guards, package boundary, and gateway watch        | Node-relevant changes              |
| `build-smoke`                    | Built-CLI smoke tests and startup-memory smoke                                                            | Node-relevant changes              |
| `checks`                         | Verifier for built-artifact channel tests                                                                 | Node-relevant changes              |
| `checks-node-compat-node22`      | Node 22 compatibility build and smoke lane                                                                | Manual CI dispatch for releases    |
| `check-docs`                     | Docs formatting, lint, and broken-link checks                                                             | Docs changed                       |
| `skills-python`                  | Ruff + pytest for Python-backed skills                                                                    | Python-skill-relevant changes      |
| `checks-windows`                 | Windows-specific process/path tests plus shared runtime import specifier regressions                      | Windows-relevant changes           |
| `macos-node`                     | macOS TypeScript test lane using the shared built artifacts                                               | macOS-relevant changes             |
| `macos-swift`                    | Swift lint, build, and tests for the macOS app                                                            | macOS-relevant changes             |
| `android`                        | Android unit tests for both flavors plus one debug APK build                                              | Android-relevant changes           |
| `test-performance-agent`         | Daily Codex slow-test optimization after trusted activity                                                 | Main CI success or manual dispatch |
| `openclaw-performance`           | Daily/on-demand Kova runtime performance reports with mock-provider, deep-profile, and GPT 5.4 live lanes | Scheduled and manual dispatch      |

## Fail-fast order

1. `preflight` decides which lanes exist at all. The `docs-scope` and `changed-scope` logic are steps inside this job, not standalone jobs.
2. `security-scm-fast`, `security-dependency-audit`, `security-fast`, `check`, `check-additional`, `check-docs`, and `skills-python` fail quickly without waiting on the heavier artifact and platform matrix jobs.
3. `build-artifacts` overlaps with the fast Linux lanes so downstream consumers can start as soon as the shared build is ready.
4. Heavier platform and runtime lanes fan out after that: `checks-fast-core`, `checks-fast-contracts-channels`, `checks-node-core-test`, `checks`, `checks-windows`, `macos-node`, `macos-swift`, and `android`.

GitHub may mark superseded jobs as `cancelled` when a newer push lands on the same PR or `main` ref. Treat that as CI noise unless the newest run for the same ref is also failing. Aggregate shard checks use `!cancelled() && always()` so they still report normal shard failures but do not queue after the whole workflow has already been superseded. The automatic CI concurrency key is versioned (`CI-v7-*`) so a GitHub-side zombie in an old queue group cannot indefinitely block newer main runs. Manual full-suite runs use `CI-manual-v1-*` and do not cancel in-progress runs.

The `ci-timings-summary` job uploads a compact `ci-timings-summary` artifact for each non-draft CI run. It records wall time, queue time, slowest jobs, and failed jobs for the current run, so CI health checks do not need to scrape the full Actions payload repeatedly.

## Scope and routing

Scope logic lives in `scripts/ci-changed-scope.mjs` and is covered by unit tests in `src/scripts/ci-changed-scope.test.ts`. Manual dispatch skips changed-scope detection and makes the preflight manifest act as if every scoped area changed.

- **CI workflow edits** validate the Node CI graph plus workflow linting, but do not force Windows, Android, or macOS native builds by themselves; those platform lanes stay scoped to platform source changes.
- **CI routing-only edits, selected cheap core-test fixture edits, and narrow plugin contract helper/test-routing edits** use a fast Node-only manifest path: `preflight`, security, and a single `checks-fast-core` task. That path skips build artifacts, Node 22 compatibility, channel contracts, full core shards, bundled-plugin shards, and additional guard matrices when the change is limited to the routing or helper surfaces the fast task exercises directly.
- **Windows Node checks** are scoped to Windows-specific process/path wrappers, npm/pnpm/UI runner helpers, package manager config, and the CI workflow surfaces that execute that lane; unrelated source, plugin, install-smoke, and test-only changes stay on the Linux Node lanes.

The slowest Node test families are split or balanced so each job stays small without over-reserving runners: channel contracts run as three weighted Blacksmith-backed shards with the standard GitHub runner fallback, core unit fast/support lanes run separately, core runtime infra is split between state, process/config, cron, and shared shards, auto-reply runs as balanced workers (with the reply subtree split into agent-runner, dispatch, and commands/state-routing shards), and agentic gateway/server configs are split across chat/auth/model/http-plugin/runtime/startup lanes instead of waiting on built artifacts. Broad browser, QA, media, and miscellaneous plugin tests use their dedicated Vitest configs instead of the shared plugin catch-all. Include-pattern shards record timing entries using the CI shard name, so `.artifacts/vitest-shard-timings.json` can distinguish a whole config from a filtered shard. `check-additional` keeps package-boundary compile/canary work together and separates runtime topology architecture from gateway watch coverage; the boundary guard list is striped across four matrix shards, each running selected independent guards concurrently and printing per-check timings. The expensive Codex happy-path prompt snapshot drift check runs as its own additional job for manual CI and for prompt-affecting changes only, so normal unrelated Node changes do not wait behind cold prompt snapshot generation and the boundary shards stay balanced while prompt drift is still pinned to the PR that caused it; the same flag skips prompt snapshot Vitest generation inside the built-artifact core support-boundary shard. Gateway watch, channel tests, and the core support-boundary shard run concurrently inside `build-artifacts` after `dist/` and `dist-runtime/` are already built.

Android CI runs both `testPlayDebugUnitTest` and `testThirdPartyDebugUnitTest` and then builds the Play debug APK. The third-party flavor has no separate source set or manifest; its unit-test lane still compiles the flavor with the SMS/call-log BuildConfig flags, while avoiding a duplicate debug APK packaging job on every Android-relevant push.

The `check-dependencies` shard runs `pnpm deadcode:dependencies` (a production Knip dependency-only pass pinned to the lat
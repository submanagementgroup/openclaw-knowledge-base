---
domain: reference
topic: "Testing Reference: Test Suite Types, Configuration, and Running Tests"
type: reference
keywords:
  - testing
  - test suite
  - unit tests
  - integration tests
  - test reference
related:
  - plugins/sdk-testing
  - help/testing
source: reference/test.md
---

OpenClaw test suite reference: test types, configuration, and running tests.

- Full testing kit (suites, live, Docker): [Testing](/help/testing)

- `pnpm test:force`: Kills any lingering gateway process holding the default control port, then runs the full Vitest suite with an isolated gateway port so server tests don’t collide with a running instance. Use this when a prior gateway run left port 18789 occupied.
- `pnpm test:coverage`: Runs the unit suite with V8 coverage (via `vitest.unit.config.ts`). This is a loaded-file unit coverage gate, not whole-repo all-file coverage. Thresholds are 70% lines/functions/statements and 55% branches. Because `coverage.all` is false, the gate measures files loaded by the unit coverage suite instead of treating every split-lane source file as uncovered.
- `pnpm test:coverage:changed`: Runs unit coverage only for files changed since `origin/main`.
- `pnpm test:changed`: cheap smart changed test run. It runs precise targets from direct test edits, sibling `*.test.ts` files, explicit source mappings, and the local import graph. Broad/config/package changes are skipped unless they map to precise tests.
- `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`: explicit broad changed test run. Use it when a test harness/config/package edit should fall back to Vitest's broader changed-test behavior.
- `pnpm changed:lanes`: shows the architectural lanes triggered by the diff against `origin/main`.
- `pnpm check:changed`: runs the smart changed check gate for the diff against `origin/main`. It runs typecheck, lint, and guard commands for the affected architectural lanes, but does not run Vitest tests. Use `pnpm test:changed` or explicit `pnpm test <target>` for test proof.
- `pnpm test`: routes explicit file/directory targets through scoped Vitest lanes. Untargeted runs use fixed shard groups and expand to leaf configs for local parallel execution; the extension group always expands to the per-extension shard configs instead of one giant root-project process.
- Test wrapper runs end with a short `[test] passed|failed|skipped ... in ...` summary. Vitest's own duration line stays the per-shard detail.
- Shared OpenClaw test state: use `src/test-utils/openclaw-test-state.ts` from Vitest when a test needs an isolated `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, config fixture, workspace, agent dir, or auth-profile store.
- Process E2E helpers: use `test/helpers/openclaw-test-instance.ts` when a Vitest process-level E2E test needs a running Gateway, CLI env, log capture, and cleanup in one place.
- Docker/Bash E2E helpers: lanes that source `scripts/lib/docker-e2e-image.sh` can pass `docker_e2e_test_state_shell_b64 <label> <scenario>` into the container and decode it with `scripts/lib/openclaw-e2e-instance.sh`; multi-home scripts can pass `docker_e2e_test_state_function_b64` and call `openclaw_test_state_create <label> <scenario>` in each flow. Lower-level callers can use `scripts/lib/openclaw-test-state.mjs shell --label <name> --scenario <name>` for an in-container shell snippet, or `node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` for a sourceable host env file. The `--` before `create` keeps newer Node runtimes from treating `--env-file` as a Node flag. Docker/Bash lanes that launch a Gateway can source `scripts/lib/openclaw-e2e-instance.sh` inside the container for entrypoint resolution, mock OpenAI startup, Gateway foreground/background launch, readiness probes, state env export, log dumps, and process cleanup.
- Full, extension, and include-pattern shard runs update local timing data in `.artifacts/vitest-shard-timings.json`; later whole-config runs use those timings to balance slow and fast shards. Include-pattern CI shards append the shard name to the timing key, which keeps filtered shard timings visible without replacing whole-config timing data. Set `OPENCLAW_TEST_PROJECTS_TIMINGS=0` to ignore the local timing artifact.
- Selected `plugin-sdk` and `commands` test files now route through dedicated light lanes that keep only `test/setup.ts`, leaving runtime-heavy cases on their existing lanes.
- Source files with sibling tests map to that sibling before falling back to wider directory globs. Helper edits under `src/channels/plugins/contracts/test-helpers`, `src/plugin-sdk/test-helpers`, and `src/plugins/contracts` use a local import graph to run importing tests instead of broad-running every shard when the dependency path is precise.
- `auto-reply` now also splits into three dedicated configs (`core`, `top-level`, `reply`) so the reply harness does not dominate the lighter top-level status/token/helper tests.
- Base Vitest config now defaults to `pool: "threads"` and `isolate: false`, with the shared non-isolated runner enabled across the repo configs.
- `pnpm test:channels` runs `vitest.channels.config.ts`.
- `pnpm test:extensions` and `pnpm test extensions` run all extension/plugin shards. Heavy channel plugins, the browser plugin, and OpenAI run as dedicated shards; other plugin groups stay batched. Use `pnpm test extensions/<id>` for one bundled plugin lane.
- `pnpm test:perf:imports`: enables Vitest import-duration + import-breakdown reporting, while still using scoped lane routing for explicit file/directory targets.
- `pnpm test:perf:imports:changed`: same import profiling, but only for files changed since `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` benchmarks the routed changed-mode path against the native root-project run for the same committed git diff.
- `pnpm test:perf:changed:bench -- --worktree` benchmarks the current worktree change set without committing first.
- `pnpm test:perf:profile:main`: writes a CPU profile for the Vitest main thread (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner`: writes CPU + heap profiles for the unit runner (`.artifacts/vitest-runner-profile`).
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`: runs every full-suite Vitest leaf config serially and writes grouped duration data plus per-config JSON/log artifacts. The Test Performance Agent uses this as its baseline before attempting slow-test fixes.
- `pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json`: compares grouped reports after a performance-focused change.
- Gateway integration: opt-in via `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` or `pnpm test:gateway`.
- `pnpm test:e2e`: Runs gateway end-to-end smoke tests (multi-instance WS/HTTP/node pairing). Defaults to `threads` + `isolate: false` with adaptive workers in `vitest.e2e.config.ts`; tune with `OPENCLAW_E2E_WORKERS=<n>` and set `OPENCLAW_E2E_VERBOSE=1` for verbose logs.
- `pnpm test:live`: Runs provider live tests (minimax/zai). Requires API keys and `LIVE=1` (or provider-specific `*_LIVE_TEST=1`) to unskip.
- `pnpm test:docker:all`: Builds the shared live-test image, packs OpenClaw once as an npm tarball, builds/reuses a bare Node/Git runner image plus a functional image that installs that tarball into `/app`, then runs Docker smoke lanes with `OPENCLAW_SKIP_DOCKER_BUILD=1` through a weighted scheduler. The bare image (`OPENCLAW_DOCKER_E2E_BARE_IMAGE`) is used for installer/update/plugin-dependency lanes; those lanes mount the prebuilt tarball instead of using copied repo sources. The functional image (`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`) is used for normal built-app functionality lanes. `scripts/package-openclaw-for-docker.mjs` is the single local/CI package packer and validates the tarball plus `dist/postinstall-inventory.json` before Docker consumes it. Docker lane definitions live in `scripts/lib/docker-e2e-scenarios.mjs`; planner logic lives in `scripts/lib/docker-e2e-plan.mjs`; `scripts/test-docker-all.mjs` executes the selected plan. `node scripts/test-docker-all.mjs --plan-json` emits the scheduler-owned CI plan for selected lanes, image kinds, package/live-image needs, state scenarios, and credential check

---
domain: reference
topic: "Local Testing Commands: vitest, test:changed, coverage, and CLI startup benchmarks"
type: reference
keywords:
  - pnpm test
  - vitest
  - test:force
  - test:coverage
  - test:changed
  - OPENCLAW_TEST_CHANGED_BROAD
  - changed:lanes
  - test:channels
  - test:extensions
  - test:perf
  - test:startup:bench
  - openclaw-test-state
  - PR gate
source: reference/test.md
related:
  - reference/testing-docker
  - troubleshooting/testing
  - troubleshooting/testing-live
---

OpenClaw testing reference for local development. Covers the `pnpm test` routing system, `vitest` lanes, changed-test detection, coverage gates, and CLI startup benchmarks.

## Core Test Commands

- `pnpm test:force` — kills any lingering gateway process holding the default control port, then runs the full Vitest suite with an isolated gateway port. Use when a prior gateway run left port 18789 occupied.
- `pnpm test:coverage` — runs the unit suite with V8 coverage via `vitest.unit.config.ts`. Thresholds are 70% lines/functions/statements and 55% branches. Because `coverage.all` is false and the default lane scopes coverage includes to non-fast unit tests with sibling source files, the gate measures source owned by this lane.
- `pnpm test:coverage:changed` — runs unit coverage only for files changed since `origin/main`.
- `pnpm test:changed` — cheap smart changed test run. Runs precise targets from direct test edits, sibling `*.test.ts` files, explicit source mappings, and the local import graph. Broad/config/package changes are skipped unless they map to precise tests.
- `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` — explicit broad changed test run. Use when a test harness/config/package edit should fall back to Vitest's broader changed-test behavior.
- `pnpm changed:lanes` — shows the architectural lanes triggered by the diff against `origin/main`.
- `pnpm check:changed` — runs the smart changed check gate for the diff against `origin/main`. Runs typecheck, lint, and guard commands for affected architectural lanes, but does not run Vitest tests.
- `pnpm test` — routes explicit file/directory targets through scoped Vitest lanes. Untargeted runs use fixed shard groups and expand to leaf configs for local parallel execution.

## Lane and Shard Organization

- `pnpm test:channels` — runs `vitest.channels.config.ts`.
- `pnpm test:extensions` and `pnpm test extensions` — run all extension/plugin shards. Heavy channel plugins, the browser plugin, and OpenAI run as dedicated shards; other plugin groups stay batched. Use `pnpm test extensions/<id>` for one bundled plugin lane.
- `pnpm test:gateway` — opt-in gateway integration tests. Also available via `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test`.
- `pnpm test:e2e` — gateway end-to-end smoke tests (multi-instance WS/HTTP/node pairing). Defaults to `threads` + `isolate: false` with adaptive workers in `vitest.e2e.config.ts`. Tune with `OPENCLAW_E2E_WORKERS=<n>`; set `OPENCLAW_E2E_VERBOSE=1` for verbose logs.
- `pnpm test:live` — provider live tests (minimax/zai). Requires API keys and `LIVE=1` (or provider-specific `*_LIVE_TEST=1`) to unskip.

## Test State Helpers

- **Shared OpenClaw test state:** use `src/test-utils/openclaw-test-state.ts` from Vitest when a test needs an isolated `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, config fixture, workspace, agent dir, or auth-profile store.
- **Process E2E helpers:** use `test/helpers/openclaw-test-instance.ts` when a Vitest process-level E2E test needs a running Gateway, CLI env, log capture, and cleanup in one place.
- **Docker/Bash E2E helpers:** lanes that source `scripts/lib/docker-e2e-image.sh` can pass `docker_e2e_test_state_shell_b64 <label> <scenario>` into the container; multi-home scripts can call `openclaw_test_state_create <label> <scenario>` in each flow. Lower-level callers can use `scripts/lib/openclaw-test-state.mjs shell --label <name> --scenario <name>` for an in-container shell snippet, or `node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` for a sourceable host env file.

## Shard Timing

Full, extension, and include-pattern shard runs update local timing data in `.artifacts/vitest-shard-timings.json`. Later whole-config runs use those timings to balance slow and fast shards. Set `OPENCLAW_TEST_PROJECTS_TIMINGS=0` to ignore the local timing artifact.

## Local PR Gate

For local PR land/gate checks, run:

```bash
pnpm check:changed
pnpm check
pnpm check:test-types
pnpm build
pnpm test
pnpm check:docs
```

If `pnpm test` flakes on a loaded host, rerun once before treating it as a regression, then isolate with `pnpm test <path/to/test>`. For memory-constrained hosts, use:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed
```

## Performance Profiling

- `pnpm test:perf:imports` — enables Vitest import-duration + import-breakdown reporting, while still using scoped lane routing for explicit file/directory targets.
- `pnpm test:perf:imports:changed` — same import profiling, but only for files changed since `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` — benchmarks the routed changed-mode path against the native root-project run for the same committed git diff.
- `pnpm test:perf:changed:bench -- --worktree` — benchmarks the current worktree change set without committing first.
- `pnpm test:perf:profile:main` — writes a CPU profile for the Vitest main thread (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner` — writes CPU + heap profiles for the unit runner (`.artifacts/vitest-runner-profile`).
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json` — runs every full-suite Vitest leaf config serially and writes grouped duration data. The Test Performance Agent uses this as its baseline before attempting slow-test fixes.
- `pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` — compares grouped reports after a performance-focused change.

## CLI Startup Bench

Script: `scripts/bench-cli-startup.ts`

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
```

Presets:
- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `tasks --json`, tasks list, tasks audit, agents list, gateway status, gateway status --json, gateway health, config get
- `all`: both presets

Output includes `sampleCount`, avg, p50, p95, min/max, exit-code/signal distribution, and max RSS summaries for each command. Optional `--cpu-prof-dir` / `--heap-prof-dir` writes V8 profiles per run.

Saved output conventions:
- `pnpm test:startup:bench:smoke` — writes smoke artifact to `.artifacts/cli-startup-bench-smoke.json`
- `pnpm test:startup:bench:save` — writes full-suite artifact to `.artifacts/cli-startup-bench-all.json` using `runs=5` and `warmup=1`
- `pnpm test:startup:bench:update` — refreshes the checked-in baseline fixture at `test/fixtures/cli-startup-bench.json`
- `pnpm test:startup:bench:check` — compares current results against the fixture

## Model Latency Bench (Local Keys)

Script: `scripts/bench-model.ts`

```bash
source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10
```

Optional env: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`.
Default prompt: "Reply with a single word: ok. No punctuation or extra text."

## Vitest Configuration Notes

- Base Vitest config defaults to `pool: "threads"` and `isolate: false`, with the shared non-isolated runner enabled across the repo configs.
- `auto-reply` splits into three dedicated configs (`core`, `top-level`, `reply`) so the reply harness does not dominate lighter top-level tests.
- Selected `plugin-sdk` and `commands` test files route through dedicated light lanes.
- Source files with sibling tests map to that sibling before falling back to wider directory globs.
- Test wrapper runs end with a short `[test] passed|failed|skipped ... in ...` summary.

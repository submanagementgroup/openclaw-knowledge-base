---
domain: reference
topic: "Docker and E2E Test Commands: test:docker:all, onboarding, upgrade, plugin, and browser test lanes"
type: reference
keywords:
  - pnpm test:docker:all
  - docker e2e
  - test:docker
  - upgrade survivor
  - onboard-docker
  - OPENCLAW_DOCKER_ALL_PARALLELISM
  - docker lane
  - test:docker:browser-cdp-snapshot
  - test:docker:mcp-channels
  - published-upgrade-survivor
  - pnpm test:docker:plugins
source: reference/test.md
related:
  - reference/testing-local
  - troubleshooting/testing
  - troubleshooting/testing-live
  - troubleshooting/testing-updates-plugins
---

Docker and E2E test reference for OpenClaw. Covers the `test:docker:all` scheduler, individual Docker lanes, upgrade survivor tests, onboarding E2E, MCP channel smoke, and QR import validation.

## test:docker:all Overview

`pnpm test:docker:all` builds the shared live-test image, packs OpenClaw once as an npm tarball, then runs Docker smoke lanes with `OPENCLAW_SKIP_DOCKER_BUILD=1` through a weighted scheduler.

Two image kinds:
- **Bare image** (`OPENCLAW_DOCKER_E2E_BARE_IMAGE`): used for installer/update/plugin-dependency lanes; mounts the prebuilt tarball.
- **Functional image** (`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`): used for normal built-app functionality lanes.

`scripts/package-openclaw-for-docker.mjs` is the single local/CI package packer and validates the tarball plus `dist/postinstall-inventory.json` before Docker consumes it.

### Parallelism Controls

| Variable | Default | Description |
|---|---|---|
| `OPENCLAW_DOCKER_ALL_PARALLELISM` | 10 | Process slots |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10 | Provider-sensitive tail pool |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT` | 9 | Heavy live lane cap |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT` | 10 | npm lane cap |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT` | 7 | Service lane cap |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` | 4 | Claude heavy lane cap |
| `OPENCLAW_DOCKER_ALL_LIVE_CODEX_LIMIT` | 4 | Codex heavy lane cap |
| `OPENCLAW_DOCKER_ALL_LIVE_GEMINI_LIMIT` | 4 | Gemini heavy lane cap |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS` | 7200000 (120m) | Fallback per-lane timeout |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000 | Start stagger to avoid Docker daemon create storms |

### Live Mode Controls

- `OPENCLAW_DOCKER_ALL_LIVE_MODE=skip` — deterministic/local lanes only; alias `pnpm test:docker:local:all`
- `OPENCLAW_DOCKER_ALL_LIVE_MODE=only` — live-provider lanes only; alias `pnpm test:docker:live:all`
- Live-only mode merges main and tail live lanes into one longest-first pool so provider buckets can pack Claude, Codex, and Gemini work together.

### Scheduler Behavior

- Stops scheduling new pooled lanes after the first failure unless `OPENCLAW_DOCKER_ALL_FAIL_FAST=0` is set.
- Lane starts staggered by 2 seconds by default.
- The runner preflights Docker, cleans stale OpenClaw E2E containers, emits active-lane status every 30 seconds (override: `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`).
- Shares provider CLI tool caches between compatible lanes.
- Retries transient live-provider failures once by default (`OPENCLAW_DOCKER_ALL_LIVE_RETRIES=<n>`).
- Stores lane timings in `.artifacts/docker-tests/lane-timings.json` for longest-first ordering on later runs.
- `OPENCLAW_DOCKER_ALL_DRY_RUN=1` — print the lane manifest without running Docker.
- `OPENCLAW_DOCKER_ALL_TIMINGS=0` — disable timing reuse.
- `node scripts/test-docker-all.mjs --plan-json` — emits the scheduler-owned CI plan without building or running Docker.
- Per-lane logs, `summary.json`, `failures.json`, and phase timings are written under `.artifacts/docker-tests/<run-id>/`; use `pnpm test:docker:timings <summary.json>` to inspect slow lanes and `pnpm test:docker:rerun <run-id|summary.json|failures.json>` for targeted rerun commands.

## Individual Docker Lanes

### Browser CDP Snapshot

```bash
pnpm test:docker:browser-cdp-snapshot
```

Builds a Chromium-backed source E2E container, starts raw CDP plus an isolated Gateway, runs `browser doctor --deep`, and verifies CDP role snapshots include link URLs, cursor-promoted clickables, iframe refs, and frame metadata.

### Skill Install

```bash
pnpm test:docker:skill-install
```

Installs the packed OpenClaw tarball in a bare Docker runner, disables `skills.install.allowUploadedArchives`, resolves a current skill slug from live ClawHub search, installs it through `openclaw skills install`, and verifies `SKILL.md`, `.clawhub/origin.json`, `.clawhub/lock.json`, and `skills info --json`.

### CLI Backend Live Docker Probes

```bash
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:codex:resume
pnpm test:docker:live-cli-backend:codex:mcp
# Claude and Gemini have matching :resume and :mcp aliases
```

### Open WebUI Integration

```bash
pnpm test:docker:openwebui
```

Starts Dockerized OpenClaw + Open WebUI, signs in through Open WebUI, checks `/api/models`, then runs a real proxied chat through `/api/chat/completions`. Requires a usable live model key, pulls an external Open WebUI image, and is not expected to be CI-stable.

### MCP Channels

```bash
pnpm test:docker:mcp-channels
```

Starts a seeded Gateway container and a second client container that spawns `openclaw mcp serve`, then verifies routed conversation discovery, transcript reads, attachment metadata, live event queue behavior, outbound send routing, and Claude-style channel + permission notifications over the real stdio bridge.

### Plugin Tests

```bash
pnpm test:docker:plugins
```

Runs install/update smoke for local path, `file:`, npm registry packages with hoisted dependencies, git moving refs, ClawHub fixtures, marketplace updates, and Claude-bundle enable/inspect.

### QR Import Smoke

```bash
pnpm test:docker:qr
```

Ensures the maintained QR runtime helper loads under the supported Docker Node runtimes (Node 24 default, Node 22 compatible).

## Upgrade Survivor Tests

### Upgrade Survivor

```bash
pnpm test:docker:upgrade-survivor
```

Installs the packed OpenClaw tarball over a dirty old-user fixture, runs package update plus non-interactive doctor without live provider or channel keys, then starts a loopback Gateway and checks that agents, channel config, plugin allowlists, workspace/session files, stale legacy plugin dependency state, startup, and RPC status survive.

### Published Upgrade Survivor

```bash
pnpm test:docker:published-upgrade-survivor
```

Installs `openclaw@latest` by default, seeds realistic existing-user files, configures that baseline, updates that published install to the packed OpenClaw tarball, runs non-interactive doctor, writes `.artifacts/upgrade-survivor/summary.json`, then starts a loopback Gateway and checks that configured intents, workspace/session files, stale plugin config, startup, `/healthz`, `/readyz`, and RPC status survive or repair cleanly.

Override options:
- `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` — override one baseline
- `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` — expand an exact local matrix (e.g., `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15`)
- `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` — add scenario fixtures including `configured-plugin-installs` and `stale-source-plugin-shadow`
- Meta baseline tokens such as `last-stable-4` or `all-since-2026.4.23` are resolved before handing exact specs to Docker lanes.

### Update Migration

```bash
pnpm test:docker:update-migration
```

Runs the published-upgrade survivor harness in the cleanup-heavy `plugin-deps-cleanup` scenario, starting at `openclaw@2026.4.23` by default. The `Update Migration` workflow expands this lane with `baselines=all-since-2026.4.23` so every stable published package from `.23` onward updates to the candidate.

## Onboarding E2E (Docker)

Docker is optional; this is only needed for containerized onboarding smoke tests.

Full cold-start flow in a clean Linux container:

```bash
scripts/e2e/onboard-docker.sh
```

This script drives the interactive wizard via a pseudo-tty, verifies config/workspace/session files, then starts the gateway and runs `openclaw health`.

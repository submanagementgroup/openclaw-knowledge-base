---
domain: troubleshooting
topic: "Help: Testing"
type: troubleshooting
keywords:
  - testing
  - troubleshooting
  - help
  - FAQ
source: help/testing.md
---

OpenClaw has three Vitest suites (unit/integration, e2e, live) and a small set
of Docker runners. This doc is a "how we test" guide:

- What each suite covers (and what it deliberately does _not_ cover).
- Which commands to run for common workflows (local, pre-push, debugging).
- How live tests discover credentials and select models/providers.
- How to add regressions for real-world model/provider issues.

> **Note:** **QA stack (qa-lab, qa-channel, live transport lanes)** is documented separately:

- [QA overview](/concepts/qa-e2e-automation) - architecture, command surface, scenario authoring.
- [Matrix QA](/concepts/qa-matrix) - reference for `pnpm openclaw qa matrix`.
- [QA channel](/channels/qa-channel) - the synthetic transport plugin used by repo-backed scenarios.

This page covers running the regular test suites and Docker/Parallels runners. The QA-specific runners section below ([QA-specific runners](#qa-specific-runners)) lists the concrete `qa` invocations and points back at the references above.


## Quick start

Most days:

- Full gate (expected before push): `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- Faster local full-suite run on a roomy machine: `pnpm test:max`
- Direct Vitest watch loop: `pnpm test:watch`
- Direct file targeting now routes extension/channel paths too: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Prefer targeted runs first when you are iterating on a single failure.
- Docker-backed QA site: `pnpm qa:lab:up`
- Linux VM-backed QA lane: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

When you touch tests or want extra confidence:

- Coverage gate: `pnpm test:coverage`
- E2E suite: `pnpm test:e2e`

When debugging real providers/models (requires real creds):

- Live suite (models + gateway tool/image probes): `pnpm test:live`
- Target one live file quietly: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- Runtime performance reports: dispatch `OpenClaw Performance` with
  `live_gpt54=true` for a real `openai/gpt-5.4` agent turn or
  `deep_profile=true` for Kova CPU/heap/trace artifacts. Daily scheduled runs
  publish mock-provider, deep-profile, and GPT 5.4 lane artifacts to
  `openclaw/clawgrit-reports` when `CLAWGRIT_REPORTS_TOKEN` is configured. The
  mock-provider report also includes source-level gateway boot, memory,
  plugin-pressure, repeated fake-model hello-loop, and CLI startup numbers.
- Docker live model sweep: `pnpm test:docker:live-models`
  - Each selected model now runs a text turn plus a small file-read-style probe.
    Models whose metadata advertises `image` input also run a tiny image turn.
    Disable the extra probes with `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` or
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` when isolating provider failures.
  - CI coverage: daily `OpenClaw Scheduled Live And E2E Checks` and manual
    `OpenClaw Release Checks` both call the reusable live/E2E workflow with
    `include_live_suites: true`, which includes separate Docker live model
    matrix jobs sharded by provider.
  - For focused CI reruns, dispatch `OpenClaw Live And E2E Checks (Reusable)`
    with `include_live_suites: true` and `live_models_only: true`.
  - Add new high-signal provider secrets to `scripts/ci-hydrate-live-auth.sh`
    plus `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` and its
    scheduled/release callers.
- Native Codex bound-chat smoke: `pnpm test:docker:live-codex-bind`
  - Runs a Docker live lane against the Codex app-server path, binds a synthetic
    Slack DM with `/codex bind`, exercises `/codex fast` and
    `/codex permissions`, then verifies a plain reply and an image attachment
    route through the native plugin binding instead of ACP.
- Codex app-server harness smoke: `pnpm test:docker:live-codex-harness`
  - Runs gateway agent turns through the plugin-owned Codex app-server harness,
    verifies `/codex status` and `/codex models`, and by default exercises image,
    cron MCP, sub-agent, and Guardian probes. Disable the sub-agent probe with
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` when isolating other Codex
    app-server failures. For a focused sub-agent check, disable the other probes:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`.
    This exits after the sub-agent probe unless
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` is set.
- Codex on-demand install smoke: `pnpm test:docker:codex-on-demand`
  - Installs the packaged OpenClaw tarball in Docker, runs OpenAI API-key
    onboarding, and verifies the Codex plugin plus `@openai/codex` dependency
    were downloaded into the managed npm root on demand.
- Live plugin tool dependency smoke: `pnpm test:docker:live-plugin-tool`
  - Packs a fixture plugin with a real `slugify` dependency, installs it through
    `npm-pack:`, verifies the dependency under the managed npm root, then asks a
    live OpenAI model to call the plugin tool and return the hidden slug.
- Crestodian rescue command smoke: `pnpm test:live:crestodian-rescue-channel`
  - Opt-in belt-and-suspenders check for the message-channel rescue command
    surface. It exercises `/crestodian status`, queues a persistent model
    change, replies `/crestodian yes`, and verifies the audit/config write path.
- Crestodian planner Docker smoke: `pnpm test:docker:crestodian-planner`
  - Runs Crestodian in a configless container with a fake Claude CLI on `PATH`
    and verifies the fuzzy planner fallback translates into an audited typed
    config write.
- Crestodian first-run Docker smoke: `pnpm test:docker:crestodian-first-run`
  - Starts from an empty OpenClaw state dir, routes bare `openclaw` to
    Crestodian, applies setup/model/agent/Discord plugin + SecretRef writes,
    validates config, and verifies audit entries. The same Ring 0 setup path is
    also covered in QA Lab by
    `pnpm openclaw qa suite --scenario crestodian-ring-zero-setup`.
- Moonshot/Kimi cost smoke: with `MOONSHOT_API_KEY` set, run
  `openclaw models list --provider moonshot --json`, then run an isolated
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`
  against `moonshot/kimi-k2.6`. Verify the JSON reports Moonshot/K2.6 and the
  assistant transcript stores normalized `usage.cost`.

> **Note:** When you only need one failing case, prefer narrowing live tests via the allowlist env vars described below.


## QA-specific runners

These commands sit beside the main test suites when you need QA-lab realism:

CI runs QA Lab in dedicated workflows. Agentic parity is nested under
`QA-Lab - All Lanes` and release validation, not a standalone PR workflow.
Broad validation should use `Full Release Validation` with
`rerun_group=qa-parity` or the release-checks QA group. Stable/default release
checks keep exhaustive live/Docker soak behind `run_release_soak=true`; the
`full` profile forces soak on. `QA-Lab - All Lanes`
runs nightly on `main` and from manual dispatch with the mock parity lane, live
Matrix lane, Convex-managed live Telegram lane, and Convex-managed live Discord
lane as parallel jobs. Scheduled QA and release checks pass Matrix
`--profile fast` explicitly, while the Matrix CLI and manual workflow input
default remain `all`; manual dispatch can shard `all` into `transport`,
`media`, `e2ee-smoke`, `e2ee-deep`, and `e2ee-cli` jobs. `OpenClaw Release
Checks` runs parity plus the fast Matrix and Telegram lanes before release
approval, using `mock-openai/gpt-5.5` for release transport checks so they stay
deterministic and avoid normal provider-plugin startup. These live transport
gateways disable memory search; memory behavior stays covered by the QA parity
suites.

Full release live media shards use
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, which already has
`ffmpeg` and `ffprobe`. Docker live model/backend shards use the shared
`ghcr.io/openclaw/openclaw-live-test:<sha>` image built once per selected
commit, then pull it with `OPENCLAW_SKIP_DOCKER_BUILD=1` instead of rebuilding
inside every shard.

- `pnpm openclaw qa suite`
  - Runs repo-backed QA scenarios directly on the host.
  - Runs multiple selected scenarios in parallel by default with isolated
    gateway workers. `qa-channel` defaults to concurrency 4 (bounded by the
    selected scenario count). Use `--concurrency <count>` to tune the worker
    count, or `--concurrency 1` for the older serial lane.
  - Exits non-zero when any scenario fails. Use `--allow-failures` when you
    want artifacts without a failing exit code.
  - Supports provider modes `live-frontier`, `mock-openai`, and `aimock`.
    `aimock` starts a local AIMock-backed provider server for experimental
    fixture and protocol-mock coverage without replacing the scenario-aware
    `mock-openai` lane.
- `pnpm test:plugins:kitchen-sink-live`
  - Runs the live OpenAI Kitchen Sink plugin gauntlet through QA Lab. It
    installs the external Kitchen Sink package, verifies the plugin SDK surface
    inventory, probes `/healthz` and `/readyz`, records gateway CPU/RSS
    evidence, runs a live OpenAI turn, and checks adversarial diagnostics.
    Requires live OpenAI auth such as `OPENAI_API_KEY`. In hydrated Testbox
    sessions it automatically sources the Testbox live-auth profile when the
    `openclaw-testbox-env` helper is present.
- `pnpm test:gateway:cpu-scenarios`
  - Runs the gateway startup bench plus a small mock QA Lab scenario pack
    (`channel-chat-baseline`, `memory-failure-fallback`,
    `gateway-restart-inflight-run`) and writes a combined CPU observation
    summary under `.artifacts/gateway-cpu-scenarios/`.
  - Flags only sustained hot CPU observations by default (`--cpu-core-warn`
    plus `--hot-wall-warn-ms`), so short startup bursts are recorded as metrics
 
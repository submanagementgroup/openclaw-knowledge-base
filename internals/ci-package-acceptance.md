---
domain: internals
topic: "CI Package Acceptance Workflow: Suite Profiles, Candidate Sources, Legacy Compat"
type: reference
keywords:
  - Package Acceptance
  - candidate sources
  - suite profiles
  - legacy compatibility
  - release-path chunks
  - tunables
  - reusable live
related:
  - internals/ci-pipeline-overview
  - internals/ci-release-workflows
  - internals/ci-package-acceptance
  - internals/ci-codeql-maintenance
source: ci.md
---

## Package Acceptance

Use `Package Acceptance` when the question is "does this installable OpenClaw package work as a product?" It is different from normal CI: normal CI validates the source tree, while package acceptance validates a single tarball through the same Docker E2E harness users exercise after install or update.

### Jobs

1. `resolve_package` checks out `workflow_ref`, resolves one package candidate, writes `.artifacts/docker-e2e-package/openclaw-current.tgz`, writes `.artifacts/docker-e2e-package/package-candidate.json`, uploads both as the `package-under-test` artifact, and prints the source, workflow ref, package ref, version, SHA-256, and profile in the GitHub step summary.
2. `docker_acceptance` calls `openclaw-live-and-e2e-checks-reusable.yml` with `ref=workflow_ref` and `package_artifact_name=package-under-test`. The reusable workflow downloads that artifact, validates the tarball inventory, prepares package-digest Docker images when needed, and runs the selected Docker lanes against that package instead of packing the workflow checkout. When a profile selects multiple targeted `docker_lanes`, the reusable workflow prepares the package and shared images once, then fans those lanes out as parallel targeted Docker jobs with unique artifacts.
3. `package_telegram` optionally calls `NPM Telegram Beta E2E`. It runs when `telegram_mode` is not `none` and installs the same `package-under-test` artifact when Package Acceptance resolved one; standalone Telegram dispatch can still install a published npm spec.
4. `summary` fails the workflow if package resolution, Docker acceptance, or the optional Telegram lane failed.

### Candidate sources

- `source=npm` accepts only `openclaw@beta`, `openclaw@latest`, or an exact OpenClaw release version such as `openclaw@2026.4.27-beta.2`. Use this for published beta/stable acceptance.
- `source=ref` packs a trusted `package_ref` branch, tag, or full commit SHA. The resolver fetches OpenClaw branches/tags, verifies the selected commit is reachable from repository branch history or a release tag, installs deps in a detached worktree, and packs it with `scripts/package-openclaw-for-docker.mjs`.
- `source=url` downloads an HTTPS `.tgz`; `package_sha256` is required.
- `source=artifact` downloads one `.tgz` from `artifact_run_id` and `artifact_name`; `package_sha256` is optional but should be supplied for externally shared artifacts.

Keep `workflow_ref` and `package_ref` separate. `workflow_ref` is the trusted workflow/harness code that runs the test. `package_ref` is the source commit that gets packed when `source=ref`. This lets the current test harness validate older trusted source commits without running old workflow logic.

### Suite profiles

- `smoke` — `npm-onboard-channel-agent`, `gateway-network`, `config-reload`
- `package` — `npm-onboard-channel-agent`, `doctor-switch`, `update-channel-switch`, `bundled-channel-deps-compat`, `plugins-offline`, `plugin-update`
- `product` — `package` plus `mcp-channels`, `cron-mcp-cleanup`, `openai-web-search-minimal`, `openwebui`
- `full` — full Docker release-path chunks with OpenWebUI
- `custom` — exact `docker_lanes`; required when `suite_profile=custom`

The `package` profile uses offline plugin coverage so published-package validation is not gated on live ClawHub availability. The optional Telegram lane reuses the `package-under-test` artifact in `NPM Telegram Beta E2E`, with the published npm spec path kept for standalone dispatches.

Release checks call Package Acceptance with `source=ref`, `package_ref=<release-ref>`, `workflow_ref=<release workflow ref>`, `suite_profile=custom`, `docker_lanes='bundled-channel-deps-compat plugins-offline'`, and `telegram_mode=mock-openai`. Release-path Docker chunks cover the overlapping package/update/plugin lanes; Package Acceptance keeps the artifact-native bundled-channel compat, offline plugin, and Telegram proof against the same resolved package tarball. Cross-OS release checks still cover OS-specific onboarding, installer, and platform behavior; package/update product validation should start with Package Acceptance. The Windows packaged and installer fresh lanes also verify that an installed package can import a browser-control override from a raw absolute Windows path. The OpenAI cross-OS agent-turn smoke defaults to `OPENCLAW_CROSS_OS_OPENAI_MODEL` when set, otherwise `openai/gpt-5.4-mini`, so the install and gateway proof stays fast and deterministic.

### Legacy compatibility windows

Package Acceptance has bounded legacy-compatibility windows for already-published packages. Packages through `2026.4.25`, including `2026.4.25-beta.*`, may use the compatibility path:

- known private QA entries in `dist/postinstall-inventory.json` may point at tarball-omitted files;
- `doctor-switch` may skip the `gateway install --wrapper` persistence subcase when the package does not expose that flag;
- `update-channel-switch` may prune missing `pnpm.patchedDependencies` from the tarball-derived fake git fixture and may log missing persisted `update.channel`;
- plugin smokes may read legacy install-record locations or accept missing marketplace install-record persistence;
- `plugin-update` may allow config metadata migration while still requiring the install record and no-reinstall behavior to stay unchanged.

The published `2026.4.26` package may also warn for local build metadata stamp files that were already shipped. Later packages must satisfy the modern contracts; the same conditions fail instead of warn or skip.

### Examples

```bash
# Validate the current beta package with product-level coverage.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# Pack and validate a release branch with the current harness.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.D \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Validate a tarball URL. SHA-256 is mandatory for source=url.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Reuse a tarball uploaded by another Actions run.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

When debugging a failed package acceptance run, start at the `resolve_package` summary to confirm the package source, version, and SHA-256. Then inspect the `docker_acceptance` child run and its Docker artifacts: `.artifacts/docker-tests/**/summary.json`, `failures.json`, lane logs, phase timings, and rerun commands. Prefer rerunning the failed package profile or exact Docker lanes instead of rerunning full release validation.


## Local Docker E2E

`pnpm test:docker:all` prebuilds one shared live-test image, packs OpenClaw once as an npm tarball, and builds two shared `scripts/e2e/Dockerfile` images:

- a bare Node/Git runner for installer/update/plugin-dependency lanes;
- a functional image that installs the same tarball into `/app` for normal functionality lanes.

Docker lane definitions live in `scripts/lib/docker-e2e-scenarios.mjs`, planner logic lives in `scripts/lib/docker-e2e-plan.mjs`, and the runner only executes the selected plan. The scheduler selects the image per lane with `OPENCLAW_DOCKER_E2E_BARE_IMAGE` and `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`, then runs lanes with `OPENCLAW_SKIP_DOCKER_BUILD=1`.

### Tunables

| Variable                               | Default | Purpose                                                                                       |
| -------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | Main-pool slot count for normal lanes.                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | Provider-sensitive tail-pool slot count.                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | Concurrent live lane cap so providers do not throttle.                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 10      | Concurrent npm install lane cap.                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | Concurrent multi-service lane cap.                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | Stagger between lane starts to avoid Docker daemon create storms; set `0` for no stagger.     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | Per-lane fallback timeout (120 minutes); selected live/tail lanes use tighter caps.           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | unset   | `1` prints the scheduler plan without running lanes.                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | unset   | Comma-separated exact lane list; skips cleanup smoke so agents can reproduce one failed lane. |

A lane heavier than its effective cap can still start from an empty pool, then runs alone until it releases capacity. The local aggregate preflights Docker, removes stale OpenClaw E2E containers, emits active-lane status, persists lane timings for longest-first ordering, and stops scheduling new pooled lanes after the first failure by default.

### Reusable live/E2E workflow

The reusable live/E2E workflow asks `scripts/test-docker-all.mjs --plan-json` which package, image kind, live image, lane, and credential coverage is required. `scripts/docker-e2e.mjs` then converts that plan into GitHub outputs and summaries. It either packs OpenClaw through `scripts/package-openclaw-for-docker.mjs`, downloads a current-run package artifact, or downloads a package artifact from `package_artifact_run_id`; validates the tarball inventory; builds and pushes package-digest-tagged bare/functional GHCR Docker E2E images through Blacksmith's Docker layer cache when the plan needs package-installed lanes; and reuses provided `docker_e2e_bare_image`/`docker_e2e_functional_image` inputs or existing package-digest images instead of rebuilding. Docker image pulls are retried with a bounded 180-second per-attempt timeout so a stuck registry/cache stream retries quickly instead of consuming most of the CI critical path.

### Release-path chunks

Release Docker coverage runs smaller chunked jobs with `OPENCLAW_SKIP_DOCKER_BUILD=1` so each chunk pulls only the image kind it needs and executes multiple lanes through the same weighted scheduler:

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | bundled-channels`

Current release Docker chunks are `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` through `plugins-runtime-install-h`, `bundled-channels-core`, `bundled-channels-update-a`, `bundled-channels-update-discord`, `bundled-channels-update-b`, and `bundled-channels-contracts`. The aggregate `bundled-channels` chunk remains available for manual one-shot reruns, and `plugins-runtime-core`, `plugins-runtime`, and `plugins-integrations` remain aggregate plugin/runtime aliases. The `install-e2e` lane alias remains the aggregate manual rerun alias for both provider installer lanes. The `bundled-channels` chunk runs split `bundled-channel-*` and `bundled-channel-update-*` lanes rather than the serial all-in-one `bundled-channel-deps` lane.

OpenWebUI is folded into `plugins-runtime-services` when full release-path coverage requests it, and keeps a standalone `openwebui` chunk only for OpenWebUI-only dispatches. Bundled-channel update lanes retry once for transient npm network failures.

Each chunk uploads `.artifacts/docker-tests/` with lane logs, timings, `summary.json`, `failures.json`, phase timings, scheduler plan JSON, slow-lane tables, and per-lane rerun commands. The workflow `docker_lanes` input runs selected lanes against the prepared images instead of the chunk jobs, which keeps failed-lane debugging bounded to one targeted Docker job and prepares, downloads, or reuses the package artifact for that run; if a selected lane is a live Docker lane, the targeted job builds the live-test image locally for that rerun. Generated per-lane GitHub rerun commands include `package_artifact_run_id`, `package_artifact_name`, and prepared image inputs when those values exist, so a failed lane can reuse the exact package and images from the failed run.

```bash
pnpm test:docker:rerun <run-id>      # download Docker artifacts and print combined/per-lane targeted rerun commands
pnpm test:docker:timings <summary>   # slow-lane and phase critical-path summaries
```

The scheduled live/E2E workflow runs the full release-path Docker suite daily.


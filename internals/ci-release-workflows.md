---
domain: internals
topic: "CI Release Workflows: Full Release Validation, Live/E2E Shards, Install Smoke, Plugin Prerelease, QA Lab"
type: reference
keywords:
  - Full Release Validation
  - Live shards
  - E2E shards
  - Install smoke
  - Plugin Prerelease
  - QA Lab
  - release workflow
  - reusable workflow
related:
  - internals/ci-pipeline-overview
  - internals/ci-release-workflows
  - internals/ci-package-acceptance
  - internals/ci-codeql-maintenance
source: ci.md
---

## Full Release Validation

`Full Release Validation` is the manual umbrella workflow for "run everything before release." It accepts a branch, tag, or full commit SHA, dispatches the manual `CI` workflow with that target, dispatches `Plugin Prerelease` for release-only plugin/package/static/Docker proof, and dispatches `OpenClaw Release Checks` for install smoke, package acceptance, Docker release-path suites, live/E2E, OpenWebUI, QA Lab parity, Matrix, and Telegram lanes. It can also run the post-publish `NPM Telegram Beta E2E` workflow when a published package spec is provided.

`release_profile` controls live/provider breadth passed into release checks:

- `minimum` keeps the fastest OpenAI/core release-critical lanes.
- `stable` adds the stable provider/backend set.
- `full` runs the broad advisory provider/media matrix.

The umbrella records the dispatched child run ids, and the final `Verify full validation` job re-checks current child run conclusions and appends slowest-job tables for each child run. If a child workflow is rerun and turns green, rerun only the parent verifier job to refresh the umbrella result and timing summary.

For recovery, both `Full Release Validation` and `OpenClaw Release Checks` accept `rerun_group`. Use `all` for a release candidate, `ci` for only the normal full CI child, `release-checks` for every release child, or a narrower group: `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live`, or `npm-telegram` on the umbrella. This keeps a failed release box rerun bounded after a focused fix.

`OpenClaw Release Checks` uses the trusted workflow ref to resolve the selected ref once into a `release-package-under-test` tarball, then passes that artifact to both the live/E2E release-path Docker workflow and the package acceptance shard. That keeps the package bytes consistent across release boxes and avoids repacking the same candidate in multiple child jobs.


## Live and E2E shards

The release live/E2E child keeps broad native `pnpm test:live` coverage, but it runs it as named shards through `scripts/test-live-shard.mjs` instead of one serial job:

- `native-live-src-agents`
- `native-live-src-gateway-core`
- provider-filtered `native-live-src-gateway-profiles` jobs
- `native-live-src-gateway-backends`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- split media audio/video shards and provider-filtered music shards

That keeps the same file coverage while making slow live provider failures easier to rerun and diagnose. The aggregate `native-live-extensions-o-z`, `native-live-extensions-media`, and `native-live-extensions-media-music` shard names remain valid for manual one-shot reruns.

The native live media shards run in `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, built by the `Live Media Runner Image` workflow. That image preinstalls `ffmpeg` and `ffprobe`; media jobs only verify the binaries before setup. Keep Docker-backed live suites on normal Blacksmith runners — container jobs are the wrong place to launch nested Docker tests.

Docker-backed live model/backend shards use a separate shared `ghcr.io/openclaw/openclaw-live-test:<sha>` image per selected commit. The live release workflow builds and pushes that image once, then the Docker live model, gateway, CLI backend, ACP bind, and Codex harness shards run with `OPENCLAW_SKIP_DOCKER_BUILD=1`. If those shards rebuild the full source Docker target independently, the release run is misconfigured and will waste wall clock on duplicate image builds.


## Install smoke

The separate `Install Smoke` workflow reuses the same scope script through its own `preflight` job. It splits smoke coverage into `run_fast_install_smoke` and `run_full_install_smoke`.

- **Fast path** runs for pull requests touching Docker/package surfaces, bundled plugin package/manifest changes, or core plugin/channel/gateway/Plugin SDK surfaces that the Docker smoke jobs exercise. Source-only bundled plugin changes, test-only edits, and docs-only edits do not reserve Docker workers. The fast path builds the root Dockerfile image once, checks the CLI, runs the agents delete shared-workspace CLI smoke, runs the container gateway-network e2e, verifies a bundled extension build arg, and runs the bounded bundled-plugin Docker profile under a 240-second aggregate command timeout (each scenario's Docker run capped separately).
- **Full path** keeps QR package install and installer Docker/update coverage for nightly scheduled runs, manual dispatches, workflow-call release checks, and pull requests that truly touch installer/package/Docker surfaces. In full mode, install-smoke prepares or reuses one target-SHA GHCR root Dockerfile smoke image, then runs QR package install, root Dockerfile/gateway smokes, installer/update smokes, and the fast bundled-plugin Docker E2E as separate jobs so installer work does not wait behind the root image smokes.

`main` pushes (including merge commits) do not force the full path; when changed-scope logic would request full coverage on a push, the workflow keeps the fast Docker smoke and leaves the full install smoke to nightly or release validation.

The slow Bun global install image-provider smoke is separately gated by `run_bun_global_install_smoke`. It runs on the nightly schedule and from the release checks workflow, and manual `Install Smoke` dispatches can opt into it, but pull requests and `main` pushes do not. QR and installer Docker tests keep their own install-focused Dockerfiles.


## Plugin Prerelease

`Plugin Prerelease` is more expensive product/package coverage, so it is a separate workflow dispatched by `Full Release Validation` or by an explicit operator. Normal pull requests, `main` pushes, and standalone manual CI dispatches keep that suite off. It balances bundled plugin tests across eight extension workers; those extension shard jobs run up to two plugin config groups at a time with one Vitest worker per group and a larger Node heap so import-heavy plugin batches do not create extra CI jobs.


## QA Lab

QA Lab has dedicated CI lanes outside the main smart-scoped workflow.

- The `Parity gate` workflow runs on matching PR changes and manual dispatch; it builds the private QA runtime and compares the mock GPT-5.5 and Opus 4.6 agentic packs.
- The `QA-Lab - All Lanes` workflow runs nightly on `main` and on manual dispatch; it fans out the mock parity gate, live Matrix lane, and live Telegram and Discord lanes as parallel jobs. Live jobs use the `qa-live-shared` environment, and Telegram/Discord use Convex leases.

Release checks run Matrix and Telegram live transport lanes with the deterministic mock provider and mock-qualified models (`mock-openai/gpt-5.5` and `mock-openai/gpt-5.5-alt`) so the channel contract is isolated from live model latency and normal provider-plugin startup. The live transport gateway disables memory search because QA parity covers memory behavior separately; provider connectivity is covered by the separate live model, native provider, and Docker provider suites.

Matrix uses `--profile fast` for scheduled and release gates, adding `--fail-fast` only when the checked-out CLI supports it. The CLI default and manual workflow input remain `all`; manual `matrix_profile=all` dispatch always shards full Matrix coverage into `transport`, `media`, `e2ee-smoke`, `e2ee-deep`, and `e2ee-cli` jobs.

`OpenClaw Release Checks` also runs the release-critical QA Lab lanes before release approval; its QA parity gate runs the candidate and baseline packs as parallel lane jobs, then downloads both artifacts into a small report job for the final parity comparison.

Do not put the PR landing path behind `Parity gate` unless the change actually touches QA runtime, model-pack parity, or a surface the parity workflow owns. For normal channel, config, docs, or unit-test fixes, treat it as an optional signal and follow the scoped CI/check evidence instead.


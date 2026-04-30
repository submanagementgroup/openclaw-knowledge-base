---
domain: internals
topic: "CI Pipeline: Job Graph, Scope Gates, Runners, and Local Equivalents"
type: reference
keywords:
  - CI
  - CI pipeline
  - CI jobs
  - continuous integration
  - CI runners
  - scope gates
related:
  - reference/testing
  - troubleshooting/testing
source: ci.md
---

OpenClaw CI/CD pipeline: job graph, scope gates, runners, and local command equivalents.

The CI runs on every push to `main` and every pull request. It uses smart scoping to skip expensive jobs when only unrelated areas changed. Manual `workflow_dispatch` runs intentionally bypass smart scoping and fan out the full normal CI graph for release candidates or broad validation, with Android lanes opt-in through `include_android` for standalone manual runs. Release-only plugin prerelease lanes live in the separate `Plugin Prerelease` workflow and run only from `Full Release Validation` or an explicit manual dispatch.

The `check-dependencies` shard runs `pnpm deadcode:dependencies`, a production Knip dependency-only pass pinned to the latest Knip version used by that script, with pnpm's minimum release age disabled for the `dlx` install. It also runs `pnpm deadcode:unused-files`, which compares Knip's production unused-file findings against `scripts/deadcode-unused-files.allowlist.mjs`. That guard fails when a PR adds a new unreviewed unused file or leaves a stale allowlist entry after cleanup, while preserving intentional dynamic plugin, generated, build, live-test, and package bridge surfaces that Knip cannot resolve statically.

`Full Release Validation` is the manual umbrella workflow for "run everything
before release." It accepts a branch, tag, or full commit SHA, dispatches the
manual `CI` workflow with that target, dispatches `Plugin Prerelease` for
release-only plugin/package/static/Docker proof, and dispatches
`OpenClaw Release Checks` for install smoke, package acceptance, Docker
release-path suites, live/E2E, OpenWebUI, QA Lab parity, Matrix, and Telegram
lanes. It can also run the post-publish `NPM Telegram Beta E2E` workflow when a
published package spec is provided. `release_profile=minimum|stable|full` controls the live/provider
breadth passed into release checks: `minimum` keeps the fastest OpenAI/core
release-critical lanes, `stable` adds the stable provider/backend set, and
`full` runs the broad advisory provider/media matrix. The umbrella records the
dispatched child run ids, and the final `Verify full validation` job re-checks
the current child run conclusions and appends slowest-job tables for each child
run. If a child workflow is rerun and turns green, rerun only the parent
verifier job to refresh the umbrella result and timing summary.

For recovery, `Full Release Validation` and `OpenClaw Release Checks` both
accept `rerun_group`. Use `all` for a release candidate, `ci` for only the
normal full CI child, `release-checks` for every release child, or a narrower
release group: `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`,
`qa-parity`, `qa-live`, or `npm-telegram` on the umbrella. This keeps a failed
release box rerun bounded after a focused fix.

The release live/E2E child keeps broad native `pnpm test:live` coverage, but it
runs it as named shards (`native-live-src-agents`,
`native-live-src-gateway-core`, provider-filtered
`native-live-src-gateway-profiles` jobs,
`native-live-src-gateway-backends`, `native-live-test`,
`native-live-extensions-a-k`, `native-live-extensions-l-n`,
`native-live-extensions-openai`, `native-live-extensions-o-z-other`,
`native-live-extensions-xai`, split media audio/video shards, and
provider-filtered music shards) through `scripts/test-live-shard.mjs` instead
of one serial job. That keeps the same file coverage while making slow live
provider failures easier to rerun and diagnose. The aggregate
`native-live-extensions-o-z`, `native-live-extensions-media`, and
`native-live-extensions-media-music` shard names remain valid for manual
one-shot reruns.

The native live media shards run in
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, built by the
`Live Media Runner Image` workflow. That image preinstalls `ffmpeg` and
`ffprobe`; media jobs only verify the binaries before setup. Keep Docker-backed
live suites on normal Blacksmith runners, because container jobs are the wrong
place to launch nested Docker tests.

Docker-backed live model/backend shards use a separate shared
`ghcr.io/openclaw/openclaw-live-test:<sha>` image per selected commit. The live
release workflow builds and pushes that image once, then the Docker live model,
gateway, CLI backend, ACP bind, and Codex harness shards run with
`OPENCLAW_SKIP_DOCKER_BUILD=1`. If those shards rebuild the full source Docker
target independently, the release run is misconfigured and will waste the wall
clock on duplicate image builds.

`OpenClaw Release Checks` uses the trusted workflow ref to resolve the selected
ref once into a `release-package-under-test` tarball, then passes that artifact
to both the live/E2E release-path Docker workflow and the package acceptance
shard. That keeps the package bytes consistent across release boxes and avoids
repacking the same candidate in multiple child jobs.

`Package Acceptance` is the side-run workflow for validating a package artifact
without blocking the release workflow. It resolves one candidate from a
published npm spec, a trusted `package_ref` built with the selected
`workflow_ref` harness, an HTTPS tarball URL with SHA-256, or a tarball artifact
from another GitHub Actions run, uploads it as `package-under-test`, then reuses
the Docker release/E2E scheduler with that tarball instead of repacking the
workflow checkout. Profiles cover smoke, package, product, full, and custom
Docker lane selections. The `package` profile uses offline plugin coverage so
published-package validation is not gated on live ClawHub availability. The
optional Telegram lane reuses the
`package-under-test` artifact in the `NPM Telegram Beta E2E` workflow, with the
published npm spec path kept for standalone dispatches.

## Package acceptance

Use `Package Acceptance` when the question is "does this installable OpenClaw
package work as a product?" It is different from normal CI: normal CI validates
the source tree, while package acceptance validates a single tarball through the
same Docker E2E harness users exercise after install or update.

The workflow has four jobs:

1. `resolve_package` checks out `workflow_ref`, resolves one package candidate,
   writes `.artifacts/docker-e2e-package/openclaw-current.tgz`, writes
   `.artifacts/docker-e2e-package/package-candidate.json`, uploads both as the
   `package-under-test` artifact, and prints the source, workflow ref, package
   ref, version, SHA-256, and profile in the GitHub step summary.
2. `docker_acceptance` calls
   `openclaw-live-and-e2e-checks-reusable.yml` with `ref=workflow_ref` and
   `package_artifact_name=package-under-test`. The reusable workflow downloads
   that artifact, validates the tarball inventory, prepares package-digest
   Docker images when needed, and runs the selected Docker lanes against that
   package instead of packing the workflow checkout. When a profile selects
   multiple targeted `docker_lanes`, the reusable workflow prepares the package
   and shared images once, then fans those lanes out as parallel targeted Docker
   jobs with unique artifacts.
3. `package_telegram` optionally calls `NPM Telegram Beta E2E`. It runs when
   `telegram_mode` is not `none` and installs the same `package-under-test`
   artifact when Package Acceptance resolved one; standalone Telegram dispatch
   can still install a published npm spec.
4. `summary` fails the workflow if package resolution, Docker acceptance, or
   the optional Telegram lane failed.

Candidate sources:

- `source=npm`: accepts only `openclaw@beta`, `openclaw@latest`, or an exact
  OpenClaw release version such as `openclaw@2026.4.27-beta.2`. Use this for
  published beta/stable acceptance.
- `source=ref`: packs a trusted `package_ref` branch, tag, or full commit SHA.
  The resolver fetches OpenClaw branches/tags, verifies the selected commit is
  reachable from repository branch history or a release tag, installs deps in a
  detached worktree, and packs it with `scripts/package-openclaw-for-docker.mjs`.
- `source=url`: downloads an HTTPS `.tgz`; `package_sha256` is required.
- `source=artifact`: downloads one `.tgz` from `artifact_run_id` and
  `artifact_name`; `package_sha256` is optional but should be supplied for
  externally shared artifacts.

Keep `workflow_ref` and `package_ref` separate. `workflow_ref` is the trusted
workflow/harness code that runs the test. `package_ref` is the source commit
that gets packed when `source=ref`. This lets the current test harness validate
older trusted source commits without running old workflow logic.

Profiles map to Docker coverage:

- `smoke`: `npm-onboard-channel-agent`, `gateway-network`, `config-reload`
- `package`: `npm-onboard-channel-agent`, `doctor-switch`,
  `update-channel-switch`, `bundled-channel-deps-compat`, `plugins-offline`,
  `plugin-update`
- `product`: `package` plus `mcp-channels`, `cron-mcp-cleanup`,
  `openai-web-search-minimal`, `openwebui`
- `full`: full Docker release-path chunks with OpenWebUI
- `custom`: exact `docker_lanes`; required when `suite_profile=custom`

Release checks call Pack

---
domain: internals
topic: "Release Process and Validation (Part 3)"
type: procedure
keywords:
  - releasing
  - release process
  - release validation
  - deploy
source: 
  - reference/RELEASING.md
  - reference/full-release-validation.md
---
## Release test boxes

`Full Release Validation` is how operators kick off all pre-release tests from
one entrypoint. For a pinned commit proof on a fast-moving branch, use the
helper so every child workflow runs from a temporary branch fixed at the target
SHA:

```bash
pnpm ci:full-release --sha <full-sha>
```

The helper pushes `release-ci/<sha>-...`, dispatches `Full Release Validation`
from that branch with `ref=<sha>`, verifies every child workflow `headSha`
matches the target, then deletes the temporary branch. This avoids proving a
newer `main` child run by accident.

For release branch or tag validation, run it from the trusted `main` workflow
ref and pass the release branch or tag as `ref`:

```bash
gh workflow run full-release-validation.yml \
  --ref main \
  -f ref=release/YYYY.M.D \
  -f provider=openai \
  -f mode=both \
  -f release_profile=stable \
  -f evidence_package_spec=openclaw@YYYY.M.D-beta.N
```

The workflow resolves the target ref, dispatches manual `CI` with
`target_ref=<release-ref>`, dispatches `OpenClaw Release Checks`, prepares a
parent `release-package-under-test` artifact for package-facing checks, and
dispatches standalone package Telegram E2E when `release_profile=full` with
`rerun_group=all` or when `release_package_spec` or
`npm_telegram_package_spec` is set. `OpenClaw Release
Checks` then fans out install smoke, cross-OS release checks, live/E2E Docker
release-path coverage when soak is enabled, Package Acceptance with Telegram
package QA, QA Lab parity, live Matrix, and live Telegram. A full run is only acceptable when the
`Full Release Validation`
summary shows `normal_ci` and `release_checks` as successful. In full/all mode,
the `npm_telegram` child must also be successful; outside full/all it is skipped
unless a published `release_package_spec` or `npm_telegram_package_spec` was
provided. The final
verifier summary includes slowest-job tables for each child run, so the release
manager can see the current critical path without downloading logs.
See [Full release validation](/reference/full-release-validation) for the
complete stage matrix, exact workflow job names, stable versus full profile
differences, artifacts, and focused rerun handles.
Child workflows are dispatched from the trusted ref that runs `Full Release
Validation`, normally `--ref main`, even when the target `ref` points at an
older release branch or tag. There is no separate Full Release Validation
workflow-ref input; choose the trusted harness by choosing the workflow run ref.
Do not use `--ref main -f ref=<sha>` for exact commit proof on moving `main`;
raw commit SHAs cannot be workflow dispatch refs, so use
`pnpm ci:full-release --sha <sha>` to create the pinned temporary branch.

Use `release_profile` to select live/provider breadth:

- `minimum`: fastest release-critical OpenAI/core live and Docker path
- `stable`: minimum plus stable provider/backend coverage for release approval
- `full`: stable plus broad advisory provider/media coverage

Use `run_release_soak=true` with `stable` when the release-blocking lanes are
green and you want the exhaustive live/E2E, Docker release-path, and
bounded published upgrade-survivor sweep before promotion. That sweep covers
the latest four stable packages plus pinned `2026.4.23` and `2026.5.2`
baselines plus older `2026.4.15` coverage, with duplicate baselines removed and
each baseline sharded into its own Docker runner job. `full` implies
`run_release_soak=true`.

`OpenClaw Release Checks` uses the trusted workflow ref to resolve the target
ref once as `release-package-under-test` and reuses that artifact in cross-OS,
Package Acceptance, and release-path Docker checks when soak runs. This keeps
all package-facing boxes on the same bytes and avoids repeated package builds.
After a beta is already on npm, set `release_package_spec=openclaw@YYYY.M.D-beta.N`
so release checks download the shipped package once, extract its build source
SHA from `dist/build-info.json`, and reuse that artifact for cross-OS,
Package Acceptance, release-path Docker, and package Telegram lanes.
The cross-OS OpenAI install smoke uses `OPENCLAW_CROSS_OS_OPENAI_MODEL` when the
repo/org variable is set, otherwise `openai/gpt-5.4`, because this lane is
proving package install, onboarding, gateway startup, and one live agent turn
rather than benchmarking the slowest default model. The broader live provider
matrix remains the place for model-specific coverage.

Use these variants depending on release stage:

```bash
# Validate an unpublished release candidate branch.
gh workflow run full-release-validation.yml \
  --ref main \
  -f ref=release/YYYY.M.D \
  -f provider=openai \
  -f mode=both \
  -f release_profile=stable

# Validate an exact pushed commit.
gh workflow run full-release-validation.yml \
  --ref main \
  -f ref=<40-char-sha> \
  -f provider=openai \
  -f mode=both

# After publishing a beta, add published-package Telegram E2E.
gh workflow run full-release-validation.yml \
  --ref main \
  -f ref=release/YYYY.M.D \
  -f provider=openai \
  -f mode=both \
  -f release_profile=full \
  -f release_package_spec=openclaw@YYYY.M.D-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.D-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

Do not use the full umbrella as the first rerun after a focused fix. If one box
fails, use the failed child workflow, job, Docker lane, package profile, model
provider, or QA lane for the next proof. Run the full umbrella again only when
the fix changed shared release orchestration or made earlier all-box evidence
stale. The umbrella's final verifier re-checks the recorded child workflow run
ids, so after a child workflow is rerun successfully, rerun only the failed
`Verify full validation` parent job.

For bounded recovery, pass `rerun_group` to the umbrella. `all` is the real
release-candidate run, `ci` runs only the normal CI child, `plugin-prerelease`
runs only the release-only plugin child, `release-checks` runs every release
box, and the narrower release groups are `install-smoke`, `cross-os`,
`live-e2e`, `package`, `qa`, `qa-parity`, `qa-live`, and `npm-telegram`.
Focused `npm-telegram` reruns require `release_package_spec` or
`npm_telegram_package_spec`; full/all runs with `release_profile=full` use the
release-checks package artifact. Focused
cross-OS reruns can add `cross_os_suite_filter=windows/packaged-upgrade` or
another OS/suite filter. QA release-check failures are advisory; a QA-only
failure does not block release validation.

### Vitest

The Vitest box is the manual `CI` child workflow. Manual CI intentionally
bypasses changed scoping and forces the normal test graph for the release
candidate: Linux Node shards, bundled-plugin shards, channel contracts, Node 22
compatibility, `check`, `check-additional`, build smoke, docs checks, Python
skills, Windows, macOS, Android, and Control UI i18n.

Use this box to answer "did the source tree pass the full normal test suite?"
It is not the same as release-path product validation. Evidence to keep:

- `Full Release Validation` summary showing the dispatched `CI` run URL
- `CI` run green on the exact target SHA
- failed or slow shard names from the CI jobs when investigating regressions
- Vitest timing artifacts such as `.artifacts/vitest-shard-timings.json` when
  a run needs performance analysis

Run manual CI directly only when the release needs deterministic normal CI but
not the Docker, QA Lab, live, cross-OS, or package boxes:

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.D
```

### Docker

The Docker box lives in `OpenClaw Release Checks` through
`openclaw-live-and-e2e-checks-reusable.yml`, plus the release-mode
`install-smoke` workflow. It validates the release candidate through packaged
Docker environments instead of only source-level tests.

Release Docker coverage includes:

- full install smoke with the slow Bun global install smoke enabled
- root Dockerfile smoke image preparation/reuse by target SHA, with QR,
  root/gateway, and installer/Bun smoke jobs running as separate install-smoke
  shards
- repository E2E lanes
- release-path Docker chunks: `core`, `package-update-openai`,
  `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`,
  `plugins-runtime-services`,
  `plugins-runtime-install-a`, `plugins-runtime-install-b`,
  `plugins-runtime-install-c`, `plugins-runtime-install-d`,
  `plugins-runtime-install-e`, `plugins-runtime-install-f`,
  `plugins-runtime-install-g`, and `plugins-runtime-install-h`
- OpenWebUI coverage inside the `plugins-runtime-services` chunk when requested
- split bundled plugin install/uninstall lanes
  `bundled-plugin-install-uninstall-0` through
  `bundled-plugin-install-uninstall-23`
- live/E2E provider suites and Docker live model coverage when release checks
  include live suites

Use Docker artifacts before rerunning. The release-path scheduler uploads
`.artifacts/docker-tests/` with lane logs, `summary.json`, `failures.json`,
phase timings, scheduler plan JSON, and rerun commands. For focused recovery,
use `docker_lanes=<lane[,lane]>` on the reusable live/E2E workflow instead of
rerunning all release chunks. Generated rerun commands include prior
`package_artifact_run_id` and prepared Docker image inputs when available, so a
failed lane can reuse the same tarball and GHCR images.

### QA Lab

The QA Lab box is also part of `OpenClaw Release Checks`. It is the agentic
behavior and channel-level release gate, separate from Vitest and Docker
package mechanics.

Release QA Lab coverage includes:

- mock parity lane comparing the OpenAI candidate lane against the Opus 4.6
  baseline using the agentic parity pack
- fast live Matrix QA profile using the `qa-live-shared` environment
- live Telegram QA lane using Convex CI credential leases
- `pnpm qa:otel:smoke` when release telemetry needs explicit local proof

Use this box to answer "does the release behave correctly in QA scenarios and
live channel flows?" Keep the artifact URLs for parity, Matrix, and Telegram
lanes when approving the release. Full Matrix coverage remains available as a
manual sharded QA-Lab run rather than the default release-critical lane.

### Package

The Package box is the installable-product gate. It is backed by
`Package Acceptance` and the resolver
`scripts/resolve-openclaw-package-candidate.mjs`. The resolver normalizes a
candidate into the `package-under-test` tarball consumed by Docker E2E, validates
the package inventory, records the package version and SHA-256, and keeps the
workflow harness ref separate from the package source ref.

Supported candidate sources:

- `source=npm`: `openclaw@beta`, `openclaw@latest`, or an exact OpenClaw release
  version
- `source=ref`: pack a trusted `package_ref` branch, tag, or full commit SHA
  with the selected `workflow_ref` harness
- `source=url`: download an HTTPS `.tgz` with required `package_sha256`
- `source=artifact`: reuse a `.tgz` uploaded by another GitHub Actions run

`OpenClaw Release Checks` runs Package Acceptance with `source=artifact`, the
prepared release package artifact, `suite_profile=custom`,
`docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor update-restart-auth plugins-offline plugin-update`,
`telegram_mode=mock-openai`. Package Acceptance keeps migration, update,
configured-auth update restart, live ClawHub skill install, stale plugin dependency cleanup, offline plugin
fixtures, plugin update, and Telegram package QA against the same resolved
tarball. Blocking release checks use the default latest published package
baseline; `run_release_soak=true` or
`release_profile=full` expands to every stable npm-published baseline from
`2026.4.23` through `latest` plus reported-issue fixtures. Use
Package Acceptance with `source=npm` for an already shipped candidate, or
`source=ref`/`source=artifact` for a SHA-backed local npm tarball before
publish. It is the GitHub-native
replacement for most of the package/update coverage that previously required
Parallels. Cross-OS release checks still matter for OS-specific onboarding,
installer, and platform behavior, but package/update product validation should
prefer Package Acceptance.

The canonical checklist for update and plugin validation is
[Testing updates and plugins](/help/testing-updates-plugins). Use it when
deciding which local, Docker, Package Acceptance, or release-check lane proves a
plugin install/update, doctor cleanup, or published-package migration change.
Exhaustive published update migration from every stable `2026.4.23+` package is
a separate manual `Update Migration` workflow, not part of Full Release CI.

Legacy package-acceptance leniency is intentionally time boxed. Packages through
`2026.4.25` may use the compatibility path for metadata gaps already published
to npm: private QA inventory entries missing from the tarball, missing
`gateway install --wrapper`, missing patch files in the tarball-derived git
fixture, missing persisted `update.channel`, legacy plugin install-record
locations, missing marketplace install-record persistence, and config metadata
migration during `plugins update`. The published `2026.4.26` package may warn
for local build metadata stamp files that were already shipped. Later packages
must satisfy the modern package contracts; those same gaps fail release
validation.

Use broader Package Acceptance profiles when the release question is about an
actual installable package:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

Common package profiles:

- `smoke`: quick package install/channel/agent, gateway network, and config
  reload lanes
- `package`: install/update/restart/plugin package contracts plus live ClawHub
  skill install proof; this is the release-check default
- `product`: `package` plus MCP channels, cron/subagent cleanup, OpenAI web
  search, and OpenWebUI
- `full`: Docker release-path chunks with OpenWebUI
- `custom`: exact `docker_lanes` list for focused reruns

For package-candidate Telegram proof, enable `telegram_mode=mock-openai` or
`telegram_mode=live-frontier` on Package Acceptance. The workflow passes the
resolved `package-under-test` tarball into the Telegram lane; the standalone
Telegram workflow still accepts a published npm spec for post-publish checks.

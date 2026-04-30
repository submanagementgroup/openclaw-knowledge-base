---
domain: internals
topic: "Release Process: Versioning, Release Checklist, and Publishing"
type: reference
keywords:
  - release
  - releasing
  - versioning
  - release process
  - publishing
related:
  - internals/ci-pipeline
source: reference/RELEASING.md
---

Release process for OpenClaw: versioning, release checklist, and publishing.

OpenClaw has three public release lanes:

- stable: tagged releases that publish to npm `beta` by default, or to npm `latest` when explicitly requested
- beta: prerelease tags that publish to npm `beta`
- dev: the moving head of `main`

## Version naming

- Stable release version: `YYYY.M.D`
  - Git tag: `vYYYY.M.D`
- Stable correction release version: `YYYY.M.D-N`
  - Git tag: `vYYYY.M.D-N`
- Beta prerelease version: `YYYY.M.D-beta.N`
  - Git tag: `vYYYY.M.D-beta.N`
- Do not zero-pad month or day
- `latest` means the current promoted stable npm release
- `beta` means the current beta install target
- Stable and stable correction releases publish to npm `beta` by default; release operators can target `latest` explicitly, or promote a vetted beta build later
- Every stable OpenClaw release ships the npm package and macOS app together;
  beta releases normally validate and publish the npm/package path first, with
  mac app build/sign/notarize reserved for stable unless explicitly requested

## Release cadence

- Releases move beta-first
- Stable follows only after the latest beta is validated
- Maintainers normally cut releases from a `release/YYYY.M.D` branch created
  from current `main`, so release validation and fixes do not block new
  development on `main`
- If a beta tag has been pushed or published and needs a fix, maintainers cut
  the next `-beta.N` tag instead of deleting or recreating the old beta tag
- Detailed release procedure, approvals, credentials, and recovery notes are
  maintainer-only

## Release operator checklist

This checklist is the public shape of the release flow. Private credentials,
signing, notarization, dist-tag recovery, and emergency rollback details stay in
the maintainer-only release runbook.

1. Start from current `main`: pull latest, confirm the target commit is pushed,
   and confirm current `main` CI is green enough to branch from it.
2. Rewrite the top `CHANGELOG.md` section from real commit history with
   `/changelog`, keep entries user-facing, commit it, push it, and rebase/pull
   once more before branching.
3. Review release compatibility records in
   `src/plugins/compat/registry.ts` and
   `src/commands/doctor/shared/deprecation-compat.ts`. Remove expired
   compatibility only when the upgrade path stays covered, or record why it is
   intentionally carried.
4. Create `release/YYYY.M.D` from current `main`; do not do normal release work
   directly on `main`.
5. Bump every required version location for the intended tag, then run the
   local deterministic preflight:
   `pnpm check:test-types`, `pnpm check:architecture`,
   `pnpm build && pnpm ui:build`, and `pnpm release:check`.
6. Run `OpenClaw NPM Release` with `preflight_only=true`. Before a tag exists,
   a full 40-character release-branch SHA is allowed for validation-only
   preflight. Save the successful `preflight_run_id`.
7. Kick off all pre-release tests with `Full Release Validation` for the
   release branch, tag, or full commit SHA. This is the one manual entrypoint
   for the four big release test boxes: Vitest, Docker, QA Lab, and Package.
8. If validation fails, fix on the release branch and rerun the smallest failed
   file, lane, workflow job, package profile, provider, or model allowlist that
   proves the fix. Rerun the full umbrella only when the changed surface makes
   prior evidence stale.
9. For beta, tag `vYYYY.M.D-beta.N`, publish with npm dist-tag `beta`, then run
   post-publish package acceptance against the published `openclaw@YYYY.M.D-beta.N`
   or `openclaw@beta` package. If a pushed or published beta needs a fix, cut
   the next `-beta.N`; do not delete or rewrite the old beta.
10. For stable, continue only after the vetted beta or release candidate has the
    required validation evidence. Stable npm publish reuses the successful
    preflight artifact via `preflight_run_id`; stable macOS release readiness
    also requires the packaged `.zip`, `.dmg`, `.dSYM.zip`, and updated
    `appcast.xml` on `main`.
11. After publish, run the npm post-publish verifier, optional standalone
    published-npm Telegram E2E when you need post-publish channel proof,
    dist-tag promotion when needed, GitHub release/prerelease notes from the
    complete matching `CHANGELOG.md` section, and the release announcement
    steps.

## Release preflight

- Run `pnpm check:test-types` before release preflight so test TypeScript stays
  covered outside the faster local `pnpm check` gate
- Run `pnpm check:architecture` before release preflight so the broader import
  cycle and architecture boundary checks are green outside the faster local gate
- Run `pnpm build && pnpm ui:build` before `pnpm release:check` so the expected
  `dist/*` release artifacts and Control UI bundle exist for the pack
  validation step
- Run the manual `Full Release Validation` workflow before release approval to
  kick off all pre-release test boxes from one entrypoint. It accepts a branch,
  tag, or full commit SHA, dispatches manual `CI`, and dispatches
  `OpenClaw Release Checks` for install smoke, package acceptance, Docker
  release-path suites, live/E2E, OpenWebUI, QA Lab parity, Matrix, and Telegram
  lanes. Provide `npm_telegram_package_spec` only after a package has been
  published and the post-publish Telegram E2E should run too. Provide
  `evidence_package_spec` when the private evidence report should prove that the
  validation matches a published npm package without forcing Telegram E2E.
  Example:
  `gh workflow run full-release-validation.yml --ref main -f ref=release/YYYY.M.D`
- Run the manual `Package Acceptance` workflow when you want side-channel proof
  for a package candidate while release work continues. Use `source=npm` for
  `openclaw@beta`, `openclaw@latest`, or an exact release version; `source=ref`
  to pack a trusted `package_ref` branch/tag/SHA with the current
  `workflow_ref` harness; `source=url` for an HTTPS tarball with a required
  SHA-256; or `source=artifact` for a tarball uploaded by another GitHub
  Actions run. The workflow resolves the candidate to
  `package-under-test`, reuses the Docker E2E release scheduler against that
  tarball, and can run Telegram QA against the same tarball with
  `telegram_mode=mock-openai` or `telegram_mode=live-frontier`.
  Example: `gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f telegram_mode=mock-openai`
  Common profiles:
  - `smoke`: install/channel/agent, gateway network, and config reload lanes
  - `package`: artifact-native package/update/plugin lanes without OpenWebUI or live ClawHub
  - `product`: package profile plus MCP channels, cron/subagent cleanup,
    OpenAI web search, and OpenWebUI
  - `full`: Docker release-path chunks with OpenWebUI
  - `custom`: exact `docker_lanes` selection for a focused rerun
- Run the manual `CI` workflow directly when you only need full normal CI
  coverage for the release candidate. Manual CI dispatches bypass changed
  scoping and force the Linux Node shards, bundled-plugin shards, channel
  contracts, Node 22 compatibility, `check`, `check-additional`, build smoke,
  docs checks, Python skills, Windows, macOS, Android, and Control UI i18n
  lanes.
  Example: `gh workflow run ci.yml --ref release/YYYY.M.D`
- Run `pnpm qa:otel:smoke` when validating release telemetry. It exercises
  QA-lab through a local OTLP/HTTP receiver and verifies the exported trace
  span names, bounded attributes, and content/identifier redaction without
  requiring Opik, Langfuse, or another external collector.
- Run `pnpm release:check` before every tagged release
- Release checks now run in a separate manual workflow:
  `OpenClaw Release Checks`
- `OpenClaw Release Checks` also runs the QA Lab mock parity gate plus the fast
  live Matrix profile and Telegram QA lane before release approval. The live
  lanes use the `qa-live-shared` environment; Telegram also uses Convex CI
  credential leases. Run the manual `QA-Lab - All Lanes` workflow with
  `matrix_profile=all` and `matrix_shards=true` when you want full Matrix
  transport, media, and E2EE inventory in parallel.
- Cross-OS install and upgrade runtime validation is part of public
  `OpenClaw Release Checks` and `Full Release Validation`, which call the
  reusable workflow
  `.github/workflows/openclaw-cross-os-release-checks-reusable.yml` directly
- This split is intentional: keep the real npm release path short,
  deterministic, and artifact-focused, while slower live checks stay in their
  own lane so they do not stall or block publish
- Secret-bearing release checks should be dispatched through `Full Release
Validation` or from the `main`/release workflow ref so workflow logic and
  secrets stay controlled
- `OpenClaw Release Checks` accepts a branch, tag, or full commit SHA as long
  as the resolved commit is reachable from an OpenClaw branch or release tag
- `OpenClaw NPM Release` validation-only preflight

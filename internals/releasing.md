---
domain: internals
topic: "Release Process and Validation"
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

## Releasing

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
5. Bump every required version location for the intended tag, then run
   `pnpm release:prep`. It refreshes plugin versions, plugin inventory, config
   schema, bundled channel config metadata, config docs baseline, plugin SDK
   exports, and plugin SDK API baseline in the right order. Commit any generated
   drift before tagging. Then run the local deterministic preflight:
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
9. For beta, tag `vYYYY.M.D-beta.N`, then run `OpenClaw Release Publish` from
   the matching `release/YYYY.M.D` branch. It verifies `pnpm plugins:sync:check`,
   dispatches all publishable plugin packages to npm and the same set to
   ClawHub in parallel, and then promotes the prepared OpenClaw npm preflight
   artifact with the matching dist-tag as soon as plugin npm publish succeeds.
   After the OpenClaw npm publish child succeeds, it creates or updates the
   matching GitHub release/prerelease page from the complete matching
   `CHANGELOG.md` section. Stable releases published to npm `latest` become the
   GitHub latest release; stable maintenance releases kept on npm `beta` are
   created with GitHub `latest=false`.
   ClawHub publishing may still be running while OpenClaw npm publishes, but the
   release publish workflow prints the child run IDs immediately. By default it
   does not wait for ClawHub after dispatching it, so OpenClaw npm availability
   is not blocked by slower ClawHub approvals or registry work; set
   `wait_for_clawhub=true` when ClawHub must block workflow completion. The
   ClawHub path retries transient CLI dependency install failures, publishes
   preview-passing plugins even when one preview cell flakes, and ends with
   registry verification for every expected plugin version so partial publishes
   remain visible and retryable. After publish, run
   `pnpm release:verify-beta -- YYYY.M.D-beta.N --openclaw-npm-run <run-id> --plugin-npm-run <run-id> --plugin-clawhub-run <run-id>`
   to verify the GitHub prerelease, npm `beta` dist-tags, npm integrity,
   published install path, ClawHub exact versions, ClawHub artifacts, and child
   workflow conclusions from one command. Add `--rerun-failed-clawhub` when the
   ClawHub sidecar failed only in retryable jobs and should be rerun in place.
   Then run the post-publish package acceptance against the published
   `openclaw@YYYY.M.D-beta.N` or
   `openclaw@beta` package. If a pushed or published prerelease needs a fix,
   cut the next matching prerelease number; do not delete or rewrite the old
   prerelease.
10. For stable, continue only after the vetted beta or release candidate has the
    required validation evidence. Stable npm publish also goes through
    `OpenClaw Release Publish`, reusing the successful preflight artifact via
    `preflight_run_id`; stable macOS release readiness also requires the
    packaged `.zip`, `.dmg`, `.dSYM.zip`, and updated `appcast.xml` on `main`.
    The private macOS publish workflow publishes the signed appcast to public
    `main` automatically after release assets verify; if branch protection blocks
    the direct push, it opens or updates an appcast PR.
11. After publish, run the npm post-publish verifier, optional standalone
    published-npm Telegram E2E when you need post-publish channel proof,
    dist-tag promotion when needed, verify the generated GitHub release page,
    and run the release announcement steps.

## Release preflight

- Run `pnpm check:test-types` before release preflight so test TypeScript stays
  covered outside the faster local `pnpm check` gate
- Run `pnpm check:architecture` before release preflight so the broader import
  cycle and architecture boundary checks are green outside the faster local gate
- Run `pnpm build && pnpm ui:build` before `pnpm release:check` so the expected
  `dist/*` release artifacts and Control UI bundle exist for the pack
  validation step
- Run `pnpm release:prep` after the root version bump and before tagging. It
  runs every deterministic release generator that commonly drifts after a
  version/config/API change: plugin versions, plugin inventory, base config
  schema, bundled channel config metadata, config docs baseline, plugin SDK
  exports, and plugin SDK API baseline. `pnpm release:check` re-runs those
  guards in check mode and reports every generated drift failure it finds in one
  pass before running package release checks.
- Run the manual `Full Release Validation` workflow before release approval to
  kick off all pre-release test boxes from one entrypoint. It accepts a branch,
  tag, or full commit SHA, dispatches manual `CI`, and dispatches
  `OpenClaw Release Checks` for install smoke, package acceptance, cross-OS
  package checks, QA Lab parity, Matrix, and Telegram lanes. Stable/default runs
  keep exhaustive live/E2E and Docker release-path soak behind
  `run_release_soak=true`; `release_profile=full` forces soak on. With
  `release_profile=full` and `rerun_group=all`, it also runs package Telegram
  E2E against the `release-package-under-test` artifact from release checks.
  Provide `release_package_spec` after publishing a beta to reuse the shipped
  npm package across release checks, Package Acceptance, and package Telegram
  E2E without rebuilding the release tarball. Provide
  `npm_telegram_package_spec` only when Telegram should use a different
  published package from the rest of release validation. Provide
  `package_acceptance_package_spec` when Package Acceptance should use a
  different published package from the release package spec. Provide
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
  `telegram_mode=mock-openai` or `telegram_mode=live-frontier`. When the
  selected Docker lanes include `published-upgrade-survivor`, the package
  artifact is the candidate and `published_upgrade_survivor_baseline` selects
  the published baseline. `update-restart-auth` uses the candidate package as
  both the installed CLI and the package-under-test so it exercises the
  candidate update command's managed restart path.
  Example: `gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai`
  Common profiles:
  - `smoke`: install/channel/agent, gateway network, and config reload lanes
  - `package`: artifact-native package/update/restart/plugin lanes without OpenWebUI or live ClawHub
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
- Run `OpenClaw Release Publish` for the mutating publish sequence after the
  tag exists. Dispatch it from `release/YYYY.M.D` (or `main` when publishing a
  main-reachable tag), pass the release tag and successful OpenClaw npm
  `preflight_run_id`, and keep the default plugin publish scope
  `all-publishable` unless you are deliberately running a focused repair. The
  workflow serializes plugin npm publish, plugin ClawHub publish, and OpenClaw
  npm publish so the core package is not published before its externalized
  plugins.
- Release checks now run in a separate manual workflow:
  `OpenClaw Release Checks`
- `OpenClaw Release Checks` also runs the QA Lab mock parity lane plus the fast
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
- `OpenClaw NPM Release` validation-only preflight also accepts the current
  full 40-character workflow-branch commit SHA without requiring a pushed tag
- That SHA path is validation-only and cannot be promoted into a real publish
- In SHA mode the workflow synthesizes `v<package.json version>` only for the
  package metadata check; real publish still requires a real release tag
- Both workflows keep the real publish and promotion path on GitHub-hosted
  runners, while the non-mutating validation path can use the larger
  Blacksmith Linux runners
- That workflow runs
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  using both `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` workflow secrets
- npm release preflight no longer waits on the separate release checks lane
- Before tagging a release candidate locally, run
  `RELEASE_TAG=vYYYY.M.D-beta.N pnpm release:fast-pretag-check`. The helper
  runs the fast release guardrails, plugin npm/ClawHub release checks, build,
  UI build, and `release:openclaw:npm:check` in the order that catches common
  approval-blocking mistakes before the GitHub publish workflow starts.
- Run `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (or the matching beta/correction tag) before approval
- After npm publish, run
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (or the matching beta/correction version) to verify the published registry
  install path in a fresh temp prefix
- After a beta publish, run `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.D-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live`
  to verify installed-package onboarding, Telegram setup, and real Telegram E2E
  against the published npm package using the shared leased Telegram credential
  pool. Local maintainer one-offs may omit the Convex vars and pass the three
  `OPENCLAW_QA_TELEGRAM_*` env credentials directly.
- To run the full post-publish beta smoke from a maintainer machine, use `pnpm release:beta-smoke -- --beta betaN`. The helper runs Parallels npm update/fresh-target validation, dispatches `NPM Telegram Beta E2E`, polls the exact workflow run, downloads the artifact, and prints the Telegram report.
- Maintainers can run the same post-publish check from GitHub Actions via the
  manual `NPM Telegram Beta E2E` workflow. It is intentionally manual-only and
  does not run on every merge.
- Maintainer release automation now uses preflight-then-promote:
  - real npm publish must pass a successful npm `preflight_run_id`
  - the real npm publish must be dispatched from the same `main` or
    `release/YYYY.M.D` branch as the successful preflight run
  - stable npm releases default to `beta`
  - stable npm publish can target `latest` explicitly via workflow input
  - token-based npm dist-tag mutation now lives in
    `openclaw/releases-private/.github/workflows/openclaw-npm-dist-tags.yml`
    for security, because `npm dist-tag add` still needs `NPM_TOKEN` while the
    public repo keeps OIDC-only publish
  - public `macOS Release` is validation-only; when a tag lives only on a
    release branch but the workflow is dispatched from `main`, set
    `public_release_branch=release/YYYY.M.D`
  - real private mac publish must pass successful private mac
    `preflight_run_id` and `validate_run_id`
  - the real publish paths promote prepared artifacts instead of rebuilding
    them again
- For stable correction releases like `YYYY.M.D-N`, the post-publish verifier
  also checks the same temp-prefix upgrade path from `YYYY.M.D` to `YYYY.M.D-N`
  so release corrections cannot silently leave older global installs on the
  base stable payload
- npm release preflight fails closed unless the tarball includes both
  `dist/control-ui/index.html` and a non-empty `dist/control-ui/assets/` payload
  so we do not ship an empty browser dashboard again
- Post-publish verification also checks that published plugin entrypoints and
  package metadata are present in the installed registry layout. A release that
  ships missing plugin runtime payloads fails the postpublish verifier and
  cannot be promoted to `latest`.
- `pnpm test:install:smoke` also enforces the npm pack `unpackedSize` budget on
  the candidate update tarball, so installer e2e catches accidental pack bloat
  before the release publish path
- If the release work touched CI planning, extension timing manifests, or
  extension test matrices, regenerate and review the planner-owned
  `plugin-prerelease-extension-shard` matrix outputs from
  `.github/workflows/plugin-prerelease.yml` before approval so release notes do
  not describe a stale CI layout
- Stable macOS release readiness also includes the updater surfaces:
  - the GitHub release must end up with the packaged `.zip`, `.dmg`, and `.dSYM.zip`
  - `appcast.xml` on `main` must point at the new stable zip after publish; the
    private macOS publish workflow commits it automatically, or opens an appcast
    PR when direct push is blocked
  - the packaged app must keep a non-debug bundle id, a non-empty Sparkle feed
    URL, and a `CFBundleVersion` at or above the canonical Sparkle build floor
    for that release version

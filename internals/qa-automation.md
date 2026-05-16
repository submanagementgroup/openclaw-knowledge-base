---
domain: internals
topic: "QA End-to-End Automation Framework"
type: concept
keywords:
  - QA automation
  - e2e testing
  - automated testing
  - QA framework
  - test harness
source: concepts/qa-e2e-automation.md
---

The private QA stack is meant to exercise OpenClaw in a more realistic,
channel-shaped way than a single unit test can.

Current pieces:

- `extensions/qa-channel`: synthetic message channel with DM, channel, thread,
  reaction, edit, and delete surfaces.
- `extensions/qa-lab`: debugger UI and QA bus for observing the transcript,
  injecting inbound messages, and exporting a Markdown report.
- `extensions/qa-matrix`, future runner plugins: live-transport adapters that
  drive a real channel inside a child QA gateway.
- `qa/`: repo-backed seed assets for the kickoff task and baseline QA
  scenarios.
- [Mantis](/concepts/mantis): before and after live verification for bugs that
  need real transports, browser screenshots, VM state, and PR evidence.

## Command surface

Every QA flow runs under `pnpm openclaw qa <subcommand>`. Many have `pnpm qa:*`
script aliases; both forms are supported.

| Command                                             | Purpose                                                                                                                                                                                                                                                                 |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | Bundled QA self-check; writes a Markdown report.                                                                                                                                                                                                                        |
| `qa suite`                                          | Run repo-backed scenarios against the QA gateway lane. Aliases: `pnpm openclaw qa suite --runner multipass` for a disposable Linux VM.                                                                                                                                  |
| `qa coverage`                                       | Print the markdown scenario-coverage inventory (`--json` for machine output).                                                                                                                                                                                           |
| `qa parity-report`                                  | Compare two `qa-suite-summary.json` files and write the agentic parity report.                                                                                                                                                                                          |
| `qa character-eval`                                 | Run the character QA scenario across multiple live models with a judged report. See [Reporting](#reporting).                                                                                                                                                            |
| `qa manual`                                         | Run a one-off prompt against the selected provider/model lane.                                                                                                                                                                                                          |
| `qa ui`                                             | Start the QA debugger UI and local QA bus (alias: `pnpm qa:lab:ui`).                                                                                                                                                                                                    |
| `qa docker-build-image`                             | Build the prebaked QA Docker image.                                                                                                                                                                                                                                     |
| `qa docker-scaffold`                                | Write a docker-compose scaffold for the QA dashboard + gateway lane.                                                                                                                                                                                                    |
| `qa up`                                             | Build the QA site, start the Docker-backed stack, print the URL (alias: `pnpm qa:lab:up`; `:fast` variant adds `--use-prebuilt-image --bind-ui-dist --skip-ui-build`).                                                                                                  |
| `qa aimock`                                         | Start only the AIMock provider server.                                                                                                                                                                                                                                  |
| `qa mock-openai`                                    | Start only the scenario-aware `mock-openai` provider server.                                                                                                                                                                                                            |
| `qa credentials doctor` / `add` / `list` / `remove` | Manage the shared Convex credential pool.                                                                                                                                                                                                                               |
| `qa matrix`                                         | Live transport lane against a disposable Tuwunel homeserver. See [Matrix QA](/concepts/qa-matrix).                                                                                                                                                                      |
| `qa telegram`                                       | Live transport lane against a real private Telegram group.                                                                                                                                                                                                              |
| `qa discord`                                        | Live transport lane against a real private Discord guild channel.                                                                                                                                                                                                       |
| `qa slack`                                          | Live transport lane against a real private Slack channel.                                                                                                                                                                                                               |
| `qa mantis`                                         | Before and after verification runner for live transport bugs, with Discord status-reactions evidence, Crabbox desktop/browser smoke, and Slack-in-VNC smoke. See [Mantis](/concepts/mantis) and [Mantis Slack Desktop Runbook](/concepts/mantis-slack-desktop-runbook). |

## Operator flow

The current QA operator flow is a two-pane QA site:

- Left: Gateway dashboard (Control UI) with the agent.
- Right: QA Lab, showing the Slack-ish transcript and scenario plan.

Run it with:

```bash
pnpm qa:lab:up
```

That builds the QA site, starts the Docker-backed gateway lane, and exposes the
QA Lab page where an operator or automation loop can give the agent a QA
mission, observe real channel behavior, and record what worked, failed, or
stayed blocked.

For faster QA Lab UI iteration without rebuilding the Docker image each time,
start the stack with a bind-mounted QA Lab bundle:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` keeps the Docker services on a prebuilt image and bind-mounts
`extensions/qa-lab/web/dist` into the `qa-lab` container. `qa:lab:watch`
rebuilds that bundle on change, and the browser auto-reloads when the QA Lab
asset hash changes.

For a local OpenTelemetry trace smoke, run:

```bash
pnpm qa:otel:smoke
```

That script starts a local OTLP/HTTP trace receiver, runs the
`otel-trace-smoke` QA scenario with the `diagnostics-otel` plugin enabled, then
decodes the exported protobuf spans and asserts the release-critical shape:
`openclaw.run`, `openclaw.harness.run`, `openclaw.model.call`,
`openclaw.context.assembled`, and `openclaw.message.delivery` must be present;
model calls must not export `StreamAbandoned` on successful turns; raw diagnostic IDs and
`openclaw.content.*` attributes must stay out of the trace. It writes
`otel-smoke-summary.json` next to the QA suite artifacts.

Observability QA stays source-checkout only. The npm tarball intentionally omits
QA Lab, so package Docker release lanes do not run `qa` commands. Use
`pnpm qa:otel:smoke` from a built source checkout when changing diagnostics
instrumentation.

For a transport-real Matrix smoke lane, run:

```bash
pnpm openclaw qa matrix --profile fast --fail-fast
```

The full CLI reference, profile/scenario catalog, env vars, and artifact layout for this lane live in [Matrix QA](/concepts/qa-matrix). At a glance: it provisions a disposable Tuwunel homeserver in Docker, registers temporary driver/SUT/observer users, runs the real Matrix plugin inside a child QA gateway scoped to that transport (no `qa-channel`), then writes a Markdown report, JSON summary, observed-events artifact, and combined output log under `.artifacts/qa-e2e/matrix-<timestamp>/`.

The scenarios cover transport behavior that unit tests cannot prove end to end: mention gating, allow-bot policies, allowlists, top-level and threaded replies, DM routing, reaction handling, inbound edit suppression, restart replay dedupe, homeserver interruption recovery, approval metadata delivery, media handling, and Matrix E2EE bootstrap/recovery/verification flows. The E2EE CLI profile also drives `openclaw matrix encryption setup` and verification commands through the same disposable homeserver before checking gateway replies.

Discord also has Mantis-only opt-in scenarios for bug reproduction. Use
`--scenario discord-status-reactions-tool-only` for the explicit status reaction
timeline, or `--scenario discord-thread-reply-filepath-attachment` to create a
real Discord thread and verify that `message.thread-reply` preserves a
`filePath` attachment. These scenarios stay out of the default live Discord lane
because they are before/after repro probes rather than broad smoke coverage.
The thread-attachment Mantis workflow can also add a logged-in Discord Web
witness video when `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` or
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` is configured in the QA
environment. That viewer profile is only for visual capture; the pass/fail
decision still comes from the Discord REST oracle.

CI uses the same command surface in `.github/workflows/qa-live-transports-convex.yml`. Scheduled and default manual runs execute the fast Matrix profile with live frontier credentials, `--fast`, and `OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS=3000`. Manual `matrix_profile=all` fans out into the five profile shards so the exhaustive catalog can run in parallel while keeping one artifact directory per shard.

For transport-real Telegram, Discord, and Slack smoke lanes:

```bash
pnpm openclaw qa telegram
pnpm openclaw qa discord
pnpm openclaw qa slack
```

They target a pre-existing real channel with two bots (driver + SUT). Required env vars, scenario lists, output artifacts, and the Convex credential pool are documented in [Telegram, Discord, and Slack QA reference](#telegram-discord-and-slack-qa-reference) below.

For a full Slack desktop VM run with VNC rescue, run:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

That command leases a Crabbox desktop/browser machine, runs the Slack live lane
inside the VM, opens Slack Web in the VNC browser, captures the desktop, and
copies `slack-qa/`, `slack-desktop-smoke.png`, and `slack-desktop-smoke.mp4`
when video capture is available back to the Mantis artifact directory. Crabbox
desktop/browser leases provide the capture tools and browser/native-build helper
packages up front, so the scenario should only install fallbacks on older
leases. Mantis reports total and per-phase timings in
`mantis-slack-desktop-smoke-report.md` so slow runs show whether time went into
lease warmup, credential acquisition, remote setup, or artifact copy. Reuse
`--lease-id <cbx_...>` after logging in to Slack Web manually through VNC;
reused leases also keep Crabbox's pnpm store cache warm. The default
`--hydrate-mode source` verifies from a source checkout and runs install/build
inside the VM. Use `--hydrate-mode prehydrated` only when the reused remote
workspace already has `node_modules` and a built `dist/`; that mode skips the
expensive install/build step and fails closed when the workspace is not ready.
With `--gateway-setup`, Mantis leaves a persistent OpenClaw Slack gateway
running inside the VM on port `38973`; without it, the command runs the normal
bot-to-bot Slack QA lane and exits after artifact capture.

The operator checklist, GitHub workflow dispatch command, evidence-comment
contract, hydrate-mode decision table, timing interpretation, and failure
handling steps live in [Mantis Slack Desktop Runbook](/concepts/mantis-slack-desktop-runbook).

For an agent/CV style desktop task, run:

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.4
```

`visual-task` leases or reuses a Crabbox desktop/browser m
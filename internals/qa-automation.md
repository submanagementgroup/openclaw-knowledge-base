---
domain: internals
topic: "QA and E2E Automation: Test Framework, Test Matrix, and CI Integration"
type: reference
keywords:
  - QA automation
  - E2E automation
  - test matrix
  - QA framework
  - automated testing
related:
  - troubleshooting/testing
  - reference/testing
source:
  - concepts/qa-e2e-automation.md
  - concepts/qa-matrix.md
---

QA and E2E automation in OpenClaw: automation frameworks, test matrix, and CI integration.

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

## Command surface

Every QA flow runs under `pnpm openclaw qa <subcommand>`. Many have `pnpm qa:*`
script aliases; both forms are supported.

| Command                                             | Purpose                                                                                                                                                                |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | Bundled QA self-check; writes a Markdown report.                                                                                                                       |
| `qa suite`                                          | Run repo-backed scenarios against the QA gateway lane. Aliases: `pnpm openclaw qa suite --runner multipass` for a disposable Linux VM.                                 |
| `qa coverage`                                       | Print the markdown scenario-coverage inventory (`--json` for machine output).                                                                                          |
| `qa parity-report`                                  | Compare two `qa-suite-summary.json` files and write the agentic parity-gate report.                                                                                    |
| `qa character-eval`                                 | Run the character QA scenario across multiple live models with a judged report. See [Reporting](#reporting).                                                           |
| `qa manual`                                         | Run a one-off prompt against the selected provider/model lane.                                                                                                         |
| `qa ui`                                             | Start the QA debugger UI and local QA bus (alias: `pnpm qa:lab:ui`).                                                                                                   |
| `qa docker-build-image`                             | Build the prebaked QA Docker image.                                                                                                                                    |
| `qa docker-scaffold`                                | Write a docker-compose scaffold for the QA dashboard + gateway lane.                                                                                                   |
| `qa up`                                             | Build the QA site, start the Docker-backed stack, print the URL (alias: `pnpm qa:lab:up`; `:fast` variant adds `--use-prebuilt-image --bind-ui-dist --skip-ui-build`). |
| `qa aimock`                                         | Start only the AIMock provider server.                                                                                                                                 |
| `qa mock-openai`                                    | Start only the scenario-aware `mock-openai` provider server.                                                                                                           |
| `qa credentials doctor` / `add` / `list` / `remove` | Manage the shared Convex credential pool.                                                                                                                              |
| `qa matrix`                                         | Live transport lane against a disposable Tuwunel homeserver. See [Matrix QA](/concepts/qa-matrix).                                                                     |
| `qa telegram`                                       | Live transport lane against a real private Telegram group.                                                                                                             |
| `qa discord`                                        | Live transport lane against a real private Discord guild channel.                                                                                                      |

## Operator flow

The current QA operator flow is a two-pane QA site:

- Left: Gateway dashboard (Control UI) with the agent.
- Right: QA Lab, showing the Slack-ish transcript and scenario plan.

Run

## QA Matrix

The Matrix QA lane runs the bundled `@openclaw/matrix` plugin against a disposable Tuwunel homeserver in Docker, with temporary driver, SUT, and observer accounts plus seeded rooms. It is the live transport-real coverage for Matrix.

This is maintainer-only tooling. Packaged OpenClaw releases intentionally omit `qa-lab`, so `openclaw qa` is only available from a source checkout. Source checkouts load the bundled runner directly — no plugin install step is needed.

For broader QA framework context, see [QA overview](/concepts/qa-e2e-automation).

## Quick start

```bash
pnpm openclaw qa matrix --profile fast --fail-fast
```

Plain `pnpm openclaw qa matrix` runs `--profile all` and does not stop on first failure. Use `--profile fast --fail-fast` for a release gate; shard the catalog with `--profile transport|media|e2ee-smoke|e2ee-deep|e2ee-cli` when running the full inventory in parallel.

## What the lane does

1. Provisions a disposable Tuwunel homeserver in Docker (default image `ghcr.io/matrix-construct/tuwunel:v1.5.1`, server name `matrix-qa.test`, port `28008`).
2. Registers three temporary users — `driver` (sends inbound traffic), `sut` (the OpenClaw Matrix account under test), `observer` (third-party traffic capture).
3. Seeds rooms required by the selected scenarios (main, threading, media, restart, secondary, allowlist, E2EE, verification DM, etc.).
4. Starts a child OpenClaw gateway with the real Matrix plugin scoped to the SUT account; `qa-channel` is not loaded in the child.
5. Runs scenarios in sequence, observing events through the driver/observer Matrix clients.
6. Tears down the homeserver, writes report and summary artifacts, then exits.

## CLI

```text
pnpm openclaw qa matrix [options]
```

### Common flags

| Flag                  | Default                                       | Description                                                                                                            |
| --------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `--profile <profile>` | `all`                                         | Scenario profile. See [Profiles](#profiles).                                                                           |
| `--fail-fast`         | off                                           | Stop after the first failed check or scenario.                                                                         |
| `--scenario <id>`     | —                                             | Run only this scenario. Repeatable. See [Scenarios](#scenarios).                                                       |
| `--output-dir <path>` | `<repo>/.artifacts/qa-e2e/matrix-<timestamp>` | Where reports, summary, observed events, and the output log are written. Relative paths resolve against `--repo-root`. |
| `--repo-root <path>`  | `process.cwd()`                               | Repository root when invoking from a neutral working directory.                                                        |
| `--sut-account <id>`  | `sut`                                         | Matrix account id inside the QA gateway config.                                                                        |

### Provider flags

The lane uses a real Matrix transport but the model provider is configurable:

| Flag                     | Default          | Description                                                                                                                               |
| ------------------------ | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `--provider-mode <mode>` | `live-frontier`  | `mock-openai` for deterministic mock dispatch or `live-frontier` for live frontier providers. The legacy alias `live-openai` still works. |
| `--model <ref>`          | provider default | Primary `provider/model` ref.                                                                                                             |
| `--alt-model <ref>`      | provider default | Alternate `provider/model` ref where scenarios switch mid-run.                                                                            |
| `--fast`                 | off              | Enable provider fast mode where supported.                                                                                                |

Matrix QA does not accept `--credential-source` or `--credential-role`. The lane provisions disposable users locally; there is no shared credential pool to lease against.

## Profiles

The selected profile decides which scenarios run.

| Profile         | Use it for

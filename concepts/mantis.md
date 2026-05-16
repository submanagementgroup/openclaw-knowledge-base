---
domain: concepts
topic: "Mantis: Task Planner and Scheduler"
type: concept
keywords:
  - Mantis
  - task planner
  - task scheduler
  - Mantis agent
  - planning
  - task management
source: concepts/mantis.md
---

Mantis is the OpenClaw end-to-end verification system for bugs that need a real
runtime, a real transport, and visible proof. It runs a scenario against a known
bad ref, captures evidence, runs the same scenario against a candidate ref, and
publishes the comparison as artifacts that a maintainer can inspect from a PR or
from a local command.

Mantis starts with Discord because Discord gives us a high-value first lane:
real bot auth, real guild channels, reactions, threads, native commands, and a
browser UI where humans can visually confirm what the transport showed.

## Goals

- Reproduce a bug from a GitHub issue or PR with the same transport shape users
  see.
- Capture a **before** artifact on the baseline ref before applying the fix.
- Capture an **after** artifact on the candidate ref after applying the fix.
- Use a deterministic oracle whenever possible, such as a Discord REST reaction
  read or channel transcript check.
- Capture screenshots when the bug has a visible UI surface.
- Run locally from an agent-controlled CLI and remotely from GitHub.
- Preserve enough machine state for VNC rescue when login, browser automation, or
  provider auth gets stuck.
- Post concise status to an operator Discord channel when the run is blocked,
  needs manual VNC help, or finishes.

## Non goals

- Mantis is not a replacement for unit tests. A Mantis run should usually become
  a smaller regression test after the fix is understood.
- Mantis is not the normal fast CI gate. It is slower, uses live credentials, and
  is reserved for bugs where the live environment matters.
- Mantis should not require a human for normal operation. Manual VNC is a rescue
  path, not the happy path.
- Mantis does not store raw secrets in artifacts, logs, screenshots, Markdown
  reports, or PR comments.

## Ownership

Mantis lives in the OpenClaw QA stack.

- OpenClaw owns the scenario runtime, transport adapters, evidence schema, and
  local CLI under `pnpm openclaw qa mantis`.
- QA Lab owns the live transport harness pieces, browser capture helpers, and
  artifact writers.
- Crabbox owns warmed Linux machines when a remote VM is needed.
- GitHub Actions owns the remote workflow entrypoint and artifact retention.
- ClawSweeper owns GitHub comment routing: parsing maintainer commands,
  dispatching the workflow, and posting the final PR comment.
- OpenClaw agents drive Mantis through Codex when a scenario needs agentic setup,
  debugging, or stuck-state reporting.

This boundary keeps transport knowledge in OpenClaw, machine scheduling in
Crabbox, and maintainer workflow glue in ClawSweeper.

## Command shape

The first local command verifies the Discord bot, guild, channel, message send,
reaction send, and artifact path:

```bash
pnpm openclaw qa mantis discord-smoke \
  --output-dir .artifacts/qa-e2e/mantis/discord-smoke
```

The local before and after runner accepts this shape:

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-status-reactions-tool-only \
  --baseline origin/main \
  --candidate HEAD \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-status-reactions
```

The runner creates detached baseline and candidate worktrees under the output
directory, installs dependencies, builds each ref, runs the scenario with
`--allow-failures`, then writes `baseline/`, `candidate/`, `comparison.json`,
and `mantis-report.md`. For the first Discord scenario, a successful verification
means baseline status is `fail` and candidate status is `pass`.

The second Discord before/after probe targets thread attachments:

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-thread-reply-filepath-attachment \
  --baseline <bug-ref> \
  --candidate <fix-ref> \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-thread-attachment
```

That scenario posts a parent message with the driver bot, creates a real Discord
thread, calls OpenClaw's `message.thread-reply` action with a repo-local
`filePath`, then polls the thread for the SUT reply and attachment filename. The
baseline screenshot shows the reply with no attachment; the candidate screenshot
shows the expected `mantis-thread-report.md` attachment.

The first VM/browser primitive is the desktop smoke:

```bash
pnpm openclaw qa mantis desktop-browser-smoke \
  --output-dir .artifacts/qa-e2e/mantis/desktop-browser
```

It leases or reuses a Crabbox desktop machine, starts a visible browser inside the
VNC session, captures the desktop, pulls artifacts back to the local output
directory, and writes the reconnect command into the report. The command defaults
to the Hetzner provider because it is the first provider with working desktop/VNC
coverage in the Mantis lane. Override it with `--provider`, `--crabbox-bin`, or
`OPENCLAW_MANTIS_CRABBOX_PROVIDER` when running against another Crabbox fleet.

Useful desktop smoke flags:

- `--lease-id <cbx_...>` or `OPENCLAW_MANTIS_CRABBOX_LEASE_ID` reuses a warmed desktop.
- `--browser-url <url>` changes the page opened in the visible browser.
- `--html-file <path>` renders a repo-local HTML artifact in the visible browser. Mantis uses this to capture the generated Discord status-reaction timeline through a real Crabbox desktop.
- `--browser-profile-dir <remote-path>` reuses a remote Chrome user-data-dir so a persistent Mantis desktop can stay logged in between runs. Use this for the long-lived Discord Web viewer profile.
- `--browser-profile-archive-env <name>` restores a base64 `.tgz` Chrome user-data-dir archive from the named environment variable before launching the browser. Use this for logged-in witnesses such as Discord Web. The default env var is `OPENCLAW_MANTIS_BROWSER_PROFILE_TGZ_B64`.
- `--video-duration <seconds>` controls the MP4 capture length. Use a longer duration for slow logged-in web apps that need time to settle.
- `--keep-lease` or `OPENCLAW_MANTIS_KEEP_VM=1` keeps a newly created passing lease open for VNC inspection. Failed runs keep the lease by default when one was created so an operator can reconnect.
- `--class`, `--idle-timeout`, and `--ttl` tune machine size and lease lifetime.

For Discord Web evidence, Mantis uses a dedicated viewer account instead of a
bot token. The live Discord API scenario remains the oracle: it creates the real
thread, sends the SUT `thread-reply`, and checks the attachment through Discord
REST. When `OPENCLAW_QA_DISCORD_CAPTURE_UI_METADATA=1` is set, the scenario also
writes a Discord Web URL artifact. When `OPENCLAW_QA_DISCORD_KEEP_THREADS=1` is
set, it leaves that thread available long enough for a logged-in browser to open
and record it.

The GitHub workflow opens the candidate thread URL in Discord Web, captures a
screenshot, records an MP4, and generates a trimmed GIF preview when Crabbox
media tooling is available. Prefer a persistent viewer profile path configured
through `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR`, because full Chrome profile
archives can outgrow GitHub's secret-size limit. For small/bootstrap profiles,
the workflow can also restore a base64 `.tgz` archive from
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64`. If neither profile source is
configured, the workflow still publishes the deterministic baseline/candidate
attachment screenshots and logs a notice that the logged-in Discord Web witness
was skipped.

The first full desktop transport primitive is the Slack desktop smoke:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --output-dir .artifacts/qa-e2e/mantis/slack-desktop \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

It leases or reuses a Crabbox desktop machine, syncs the current checkout into
the VM, runs `pnpm openclaw qa slack` inside that VM, opens Slack Web in the VNC
browser, captures the visible desktop, and copies both the Slack QA artifacts and
the VNC screenshot back to the local output directory. This is the first Mantis
shape where the SUT OpenClaw gateway and the browser both live inside the same
Linux desktop VM.

With `--gateway-setup`, the command prepares a persistent disposable OpenClaw
home at `$HOME/.openclaw-mantis/slack-openclaw`, patches Slack Socket Mode
configuration for the selected channel, starts `openclaw gateway run` on port
`38973`, and keeps Chrome running in the VNC session. This is the "leave me a
Linux desktop with Slack and a claw running" mode; the bot-to-bot Slack QA lane
remains the default when `--gateway-setup` is omitted.

Required inputs for `--credential-source env`:

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`
- `OPENCLAW_LIVE_OPENAI_KEY` for the remote model lane. If only
  `OPENAI_API_KEY` is set locally, Mantis maps it to `OPENCLAW_LIVE_OPENAI_KEY`
  before invoking Crabbox so Crabbox's `OPENCLAW_*` env forwarding can carry it
  into the VM.

With `--gateway-setup --credential-source convex`, Mantis leases the Slack SUT
credential from the shared pool before creating the VM and forwards the leased
channel id, Socket Mode app token, and bot token as the `OPENCLAW_MANTIS_SLACK_*`
runtime env inside the desktop. That keeps GitHub workflows thin: they only need
the Convex broker secret, not raw Slack bot or app tokens.

Useful Slack desktop flags:

- `--lease-id <cbx_...>` reruns against a machine where an operator already logged in to Slack Web through VNC.
- `--gateway-setup` starts a persistent OpenClaw Slack gateway in the VM instead of only running the bot-to-bot QA lane.
- `--keep-lease` keeps the gateway VM open for VNC inspection after success; `--no-keep-lease` stops it after collecting artifacts.
- `--slack-url <url>` opens a specific Slack Web URL. Without it, Mantis derives `https://app.slack.com/client/<team>/<channel>` from Slack `auth.test` when the SUT bot token is available.
- `--slack-channel-id <id>` controls the Slack channel allowlist used by gateway setup.
- `OPENCLAW_MANTIS_SLACK_BROWSER_PROFILE_DIR` controls the persistent Chrome profile inside the VM. The default is `$HOME/.config/openclaw-mantis/slack-chrome-profile`, so a manual Slack Web login survives reruns on the same lease.
- `--credential-source convex --credential-role ci` uses the shared credential pool instead of direct Slack env tokens.
- `--provider-mode`, `--model`, `--alt-model`, and `--fast` pass through to the Slack live lane.

The GitHub smoke workflow is `Mantis Discord Smoke`. The before and after GitHub
workflow for the first real scenario is `Mantis Discord Status Reactions`. It
accepts:

- `baseline_ref`: the ref expected to reproduce queued-only behavior.
- `candidate_ref`: the ref expected to show `queued -> thinking -> done`.

It checks out the workflow harness ref, builds separate baseline and candidate
worktrees, runs `discord-status-reactions-tool-only` against each worktree, and
uploads `baseline/`, `candidate/`, `comparison.json`, and `mantis-report.md` as
Actions artifacts. It also renders each lane's timeline HTML in a Crabbox
desktop browser and publishes those VNC screenshots beside the deterministic
timeline PNGs in the PR comment. The same PR comment embeds lightweight
motion-trimmed GIF previews generated by `crabbox media preview`, links to the
matching motion-trimmed MP4 clips, and keeps the full desktop MP4 files for deep
inspection. Screenshots stay inline for quick review. The workflow builds the
Crabbox CLI from
`openclaw/crabbox` main so it can use the current desktop/browser lease flags
before the next Crabbox binary release is cut.

`Mantis Scenario` is the generic manual entrypoint. It takes a `scenario_id`,
`candidate_ref`, optional `baseline_ref`, and optional `pr_number`, then
dispatches the scenario-owned workflow. The wrapper is intentionally thin:
scenario workflows still own their transport setup, credentials, VM class,
expected oracle, and artifact manifest.

`Mantis Slack Desktop Smoke` is the first Slack VM workflow. It checks out the
trusted candidate ref in a separate worktree, leases a Crabbox Linux desktop,
runs `pnpm openclaw qa mantis slack-desktop-smoke --gateway-setup` against that
candidate, opens Slack Web in the VNC browser, records the desktop, generates a
motion-trimmed preview with `crabbox media preview`, uploads the full artifact
directory, and optionally posts the inline evidence comment on the target PR.
It defaults to AWS for the desktop lease and exposes a manual provider input so
operators can switch to Hetzner when AWS capacity is slow or unavailable. Use
this lane when you want "a Linux desktop with Slack and a claw running" instead
of only a bot-to-bot Slack transcript.

`Mantis Telegram Live` wraps the existing Telegram live QA lane in the same PR
evidence pipeline. It checks out the trusted candidate ref in a separate
worktree, runs `pnpm openclaw qa telegram --credential-source convex
--credential-role ci`, writes a `mantis-evidence.json` manifest from the
Telegram QA summary and observed-message artifact, renders the redacted
transcript HTML through a Crabbox desktop browser, generates a motion-trimmed GIF
with `crabbox media preview`, and posts the inline PR evidence comment when a PR
number is available. This lane is transcript-visual rather than logged-in
Telegram Web proof: the Telegram Bot API gives stable live message evidence, but
Telegram Web login state is not required for normal Mantis automation.

`Mantis Telegram Desktop Proof` is the agentic native Telegram Desktop
before/after wrapper. A maintainer can trigger it from a PR comment with
`@Mantis telegram desktop proof`, from the Actions UI with freeform
instructions, or through the generic `Mantis Scenario` dispatcher. The workflow
hands the PR, baseline ref, candidate ref, and maintainer instructions to Codex.
The agent reads the PR, decides what Telegram-visible behavior proves the
change, runs the real-user Crabbox Telegram Desktop proof lane for baseline and
candidate, iterates until the native GIFs are useful, writes paired
`motionPreview` 
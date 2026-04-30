---
domain: troubleshooting
topic: "Help Index and Scripts: Helper Scripts and Utility Reference"
type: reference
keywords:
  - scripts
  - helper scripts
  - utilities
  - help index
related:
  - troubleshooting/general-troubleshooting
source:
  - help/scripts.md
  - help/index.md
---

Helper scripts and utilities for OpenClaw development and operations.

Quick "get unstuck" path for the most common problems:

- [Troubleshooting](/help/troubleshooting) — symptom-first decision tree
- [Debugging](/help/debugging) — watch mode, raw streams, dev profile
- [Install sanity](/install/node#troubleshooting) — Node / npm / PATH checks
- [Gateway troubleshooting](/gateway/troubleshooting) — gateway-specific issues
- [Doctor](/gateway/doctor) — automated repair + diagnostic bundle

## FAQ

- [FAQ](/help/faq) — day-to-day concepts and operational questions
- [First-run FAQ](/help/faq-first-run) — install, onboard, auth, subscriptions, early failures
- [Models FAQ](/help/faq-models) — model selection, failover, auth profiles

## Diagnostics

- [Environment variables](/help/environment) — where OpenClaw loads env vars and precedence
- [Diagnostics flags](/diagnostics/flags) — runtime diagnostics and verbose modes
- [Node + tsx crash](/debug/node-issue) — specific Node / tsx runtime crash scenarios

## Testing

- [Testing](/help/testing) — test suites and Docker runners
- [Live tests](/help/testing-live) — network-touching provider and CLI smokes

## Community and meta

- [OpenClaw lore](/start/lore) — the story
- [Docs hubs](/start/hubs) — how this documentation is organized
- [Docs directory](/start/docs-directory) — full file map

## Scripts Reference

The `scripts/` directory contains helper scripts for local workflows and ops tasks.
Use these when a task is clearly tied to a script; otherwise prefer the CLI.

## Conventions

- Scripts are **optional** unless referenced in docs or release checklists.
- Prefer CLI surfaces when they exist (example: auth monitoring uses `openclaw models status --check`).
- Assume scripts are host‑specific; read them before running on a new machine.

## Auth monitoring scripts

Auth monitoring is covered in [Authentication](/gateway/authentication). The scripts under `scripts/` are optional extras for systemd/Termux phone workflows.

## GitHub read helper

Use `scripts/gh-read` when you want `gh` to use a GitHub App installation token for repo-scoped read calls while leaving normal `gh` on your personal login for write actions.

Required env:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

Optional env:

- `OPENCLAW_GH_READ_INSTALLATION_ID` when you want to skip repo-based installation lookup
- `OPENCLAW_GH_READ_PERMISSIONS` as a comma-separated override for the read permission subset to request

Repo resolution order:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

Examples:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## When adding scripts

- Keep scripts focused and documented.
- Add a short entry in the relevant doc (or create one if missing).

## Related

- [Testing](/help/testing)
- [Testing live](/help/testing-live)

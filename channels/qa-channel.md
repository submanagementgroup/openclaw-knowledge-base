---
domain: channels
topic: "QA Channel: Synthetic Transport for Automated OpenClaw Testing"
type: integration
keywords:
  - qa-channel
  - QA
  - synthetic channel
  - qa-lab
  - pnpm qa:e2e
  - baseUrl
  - botUserId
  - qa:lab:up
  - test automation
  - openclaw qa suite
  - deterministic testing
source: channels/qa-channel.md
related:
  - channels/overview
  - channels/pairing
  - channels/groups
  - reference/testing-local
  - reference/testing-docker
---

`qa-channel` is a bundled synthetic message transport for automated OpenClaw QA. It is not a production channel — it exists to exercise the same channel plugin boundary used by real transports while keeping state deterministic and fully inspectable.

## What It Does

- Slack-class target grammar: `dm:<user>`, `channel:<room>`, `group:<room>`, `thread:<room>/<thread>`
- Shared `channel:` and `group:` conversations are surfaced to agents as group/channel room turns, exercising the same visible-reply and message-tool routing policy used by Discord, Slack, Telegram, and similar transports.
- HTTP-backed synthetic bus for inbound message injection, outbound transcript capture, thread creation, reactions, edits, deletes, and search/read actions.
- Host-side self-check runner that writes a Markdown report to `.artifacts/qa-e2e/`.

## Config

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

### Account Keys

- `enabled` — master toggle for this account.
- `name` — optional display label.
- `baseUrl` — synthetic bus URL.
- `botUserId` — Matrix-style bot user id used in target grammar.
- `botDisplayName` — display name for outbound messages.
- `pollTimeoutMs` — long-poll wait window. Integer between 100 and 30000.
- `allowFrom` — sender allowlist (user ids or `"*"`). Direct messages and allowlisted group policy both use these synthetic sender ids.
- `groupPolicy` — shared-room policy: `"open"` (default), `"allowlist"`, or `"disabled"`.
- `groupAllowFrom` — optional shared-room sender allowlist. When omitted under `"allowlist"`, QA Channel falls back to `allowFrom`.
- `groups.<room>.requireMention` — require a bot mention before replying in a specific group/channel room. `groups."*"` sets the default.
- `defaultTo` — fallback target when none is supplied.
- `actions.messages` / `actions.reactions` / `actions.search` / `actions.threads` — per-action tool gating.

### Multi-Account Keys

- `accounts` — record of named per-account overrides keyed by account id.
- `defaultAccount` — preferred account id when multiple are configured.

## Runners

### Host-Side Self-Check

```bash
pnpm qa:e2e
```

Routes through `qa-lab`, starts the in-repo QA bus, boots the bundled `qa-channel` runtime slice, and runs a deterministic self-check. Writes a Markdown report under `.artifacts/qa-e2e/`.

### Full Repo-Backed Scenario Suite

```bash
pnpm openclaw qa suite
```

Runs scenarios in parallel against the QA gateway lane. See the QA overview for scenarios, profiles, and provider modes.

### Docker-Backed QA Site

```bash
pnpm qa:lab:up
```

Builds the QA site, starts the Docker-backed gateway + QA Lab stack, and prints the QA Lab URL. From there you can pick scenarios, choose the model lane, launch individual runs, and watch results live. The QA Lab debugger is separate from the shipped Control UI bundle.

---
domain: cli
topic: "CLI Commands: voicecall, webhooks, wiki..."
type: reference
keywords:
  - voicecall
  - webhooks
  - wiki
  - CLI commands
  - openclaw CLI
source: 
  - cli/voicecall.md
  - cli/webhooks.md
  - cli/wiki.md
---

## openclaw voicecall Command

# `openclaw voicecall`

`voicecall` is a plugin-provided command. It only appears when the voice-call plugin is installed and enabled.

When the Gateway is running, operational commands (`call`, `start`, `continue`, `speak`, `dtmf`, `end`, `status`) are routed to that Gateway's voice-call runtime. If no Gateway is reachable, they fall back to a standalone CLI runtime.

## Subcommands

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| Subcommand | Description                                                     |
| ---------- | --------------------------------------------------------------- |
| `setup`    | Show provider and webhook readiness checks.                     |
| `smoke`    | Run readiness checks; place a live test call only with `--yes`. |
| `call`     | Initiate an outbound voice call.                                |
| `start`    | Alias for `call` with `--to` required and `--message` optional. |
| `continue` | Speak a message and wait for the next response.                 |
| `speak`    | Speak a message without waiting for a response.                 |
| `dtmf`     | Send DTMF digits to an active call.                             |
| `end`      | Hang up an active call.                                         |
| `status`   | Inspect active calls (or one by `--call-id`).                   |
| `tail`     | Tail `calls.jsonl` (useful during provider tests).              |
| `latency`  | Summarize turn-latency metrics from `calls.jsonl`.              |
| `expose`   | Toggle Tailscale serve/funnel for the webhook endpoint.         |

## Setup and smoke

### `setup`

Prints human-readable readiness checks by default. Pass `--json` for scripts.

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

Runs the same readiness checks. It will not place a real phone call unless both `--to` and `--yes` are present.

| Flag               | Default                           | Description                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | (none)                            | Phone number to call for a live smoke.  |
| `--message <text>` | `OpenClaw voice call smoke test.` | Message to speak during the smoke call. |
| `--mode <mode>`    | `notify`                          | Call mode: `notify` or `conversation`.  |
| `--yes`            | `false`                           | Actually place the live outbound call.  |
| `--json`           | `false`                           | Print machine-readable JSON.            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # dry run
openclaw voicecall smoke --to "+15555550123" --yes  # live notify call
```

> **Note:** For external providers (`twilio`, `telnyx`, `plivo`), `setup` and `smoke` require a public webhook URL from `publicUrl`, a tunnel, or Tailscale exposure. A loopback or private serve fallback is rejected because carriers cannot reach it.


## Call lifecycle

### `call`

Initiate an outbound voice call.

| Flag                   | Required | Default           | Description                                                                |
| ---------------------- | -------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | yes      | (none)            | Message to speak when the call connects.                                   |
| `-t, --to <phone>`     | no       | config `toNumber` | E.164 phone number to call.                                                |
| `--mode <mode>`        | no       | `conversation`    | Call mode: `notify` (hang up after message) or `conversation` (stay open). |

```bash
openclaw voicecall call --to "+15555550123" --message "Hello"
openclaw voicecall call -m "Heads up" --mode notify
```

### `start`

Alias for `call` with a different default flag shape.

| Flag               | Required | Default        | Description                              |
| ------------------ | -------- | -------------- | ---------------------------------------- |
| `--to <phone>`     | yes      | (none)         | Phone number to call.                    |
| `--message <text>` | no       | (none)         | Message to speak when the call connects. |
| `--mode <mode>`    | no       | `conversation` | Call mode: `notify` or `conversation`.   |

### `continue`

Speak a message and wait for a response.

| Flag               | Required | Description       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | yes      | Call ID.          |
| `--message <text>` | yes      | Message to speak. |

### `speak`

Speak a message without waiting for a response.

| Flag               | Required | Description       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | yes      | Call ID.          |
| `--message <text>` | yes      | Message to speak. |

### `dtmf`

Send DTMF digits to an active call.

| Flag                | Required | Description                               |
| ------------------- | -------- | ----------------------------------------- |
| `--call-id <id>`    | yes      | Call ID.                                  |
| `--digits <digits>` | yes      | DTMF digits (e.g. `ww123456#` for waits). |

### `end`

Hang up an active call.

| Flag             | Required | Description |
| ---------------- | -------- | ----------- |
| `--call-id <id>` | yes      | Call ID.    |

### `status`

Inspect active calls.

| Flag             | Default | Description                  |
| ---------------- | ------- | ---------------------------- |
| `--call-id <id>` | (none)  | Restrict output to one call. |
| `--json`         | `false` | Print machine-readable JSON. |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## Logs and metrics

### `tail`

Tail the voice-call JSONL log. Prints the last `--since` lines on start, then streams new lines as they are written.

| Flag            | Default                    | Description                    |
| --------------- | -------------------------- | ------------------------------ |
| `--file <path>` | resolved from plugin store | Path to `calls.jsonl`.         |
| `--since <n>`   | `25`                       | Lines to print before tailing. |
| `--poll <ms>`   | `250` (minimum 50)         | Poll interval in milliseconds. |

### `latency`

Summarize turn-latency and listen-wait metrics from `calls.jsonl`. Output is JSON with `recordsScanned`, `turnLatency`, and `listenWait` summaries.

| Flag            | Default                    | Description                          |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | resolved from plugin store | Path to `calls.jsonl`.               |
| `--last <n>`    | `200` (minimum 1)          | Number of recent records to analyze. |

## Exposing webhooks

### `expose`

Enable, disable, or change the Tailscale serve/funnel configuration for the voice webhook.

| Flag                  | Default                                   | Description                                     |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`, `serve` (tailnet), or `funnel` (public). |
| `--path <path>`       | config `tailscale.path` or `--serve-path` | Tailscale path to expose.                       |
| `--port <port>`       | config `serve.port` or `3334`             | Local webhook port.                             |
| `--serve-path <path>` | config `serve.path` or `/voice/webhook`   | Local webhook path.                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

> **Note:** Only expose the webhook endpoint to networks you trust. Prefer Tailscale Serve over Funnel when possible.


## Related

- [CLI reference](/cli)
- [Voice call plugin](/plugins/voice-call)


## openclaw webhooks Command

# `openclaw webhooks`

Webhook helpers and integrations. Today this surface is scoped to Gmail Pub/Sub flows that integrate with the bundled `gog` watcher.

## Subcommands

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| Subcommand    | Description                                                                                  |
| ------------- | -------------------------------------------------------------------------------------------- |
| `gmail setup` | Configure Gmail watch, Pub/Sub topic/subscription, and the OpenClaw webhook delivery target. |
| `gmail run`   | Run `gog watch serve` plus the watch auto-renew loop.                                        |

## `webhooks gmail setup`

Configure Gmail watch, Pub/Sub, and OpenClaw webhook delivery.

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

### Required

| Flag                | Description             |
| ------------------- | ----------------------- |
| `--account <email>` | Gmail account to watch. |

### Pub/Sub options

| Flag                    | Default                | Description                                          |
| ----------------------- | ---------------------- | ---------------------------------------------------- |
| `--project <id>`        | (none)                 | GCP project id (the OAuth client owner).             |
| `--topic <name>`        | `gog-gmail-watch`      | Pub/Sub topic name.                                  |
| `--subscription <name>` | `gog-gmail-watch-push` | Pub/Sub subscription name.                           |
| `--label <label>`       | `INBOX`                | Gmail label to watch.                                |
| `--push-endpoint <url>` | (none)                 | Explicit Pub/Sub push endpoint. Overrides Tailscale. |

### OpenClaw delivery options

| Flag                   | Default | Description                                |
| ---------------------- | ------- | ------------------------------------------ |
| `--hook-url <url>`     | (none)  | OpenClaw webhook URL.                      |
| `--hook-token <token>` | (none)  | OpenClaw webhook token.                    |
| `--push-token <token>` | (none)  | Push token forwarded to `gog watch serve`. |

### `gog watch serve` options

| Flag                  | Default         | Description                                                       |
| --------------------- | --------------- | ----------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | `gog watch serve` bind host.                                      |
| `--port <port>`       | `8788`          | `gog watch serve` port.                                           |
| `--path <path>`       | `/gmail-pubsub` | `gog watch serve` path.                                           |
| `--include-body`      | `true`          | Include email body snippets. Pass `--no-include-body` to disable. |
| `--max-bytes <n>`     | `20000`         | Max bytes per body snippet.                                       |
| `--renew-minutes <n>` | `720` (12h)     | Renew Gmail watch every N minutes.                                |

### Tailscale exposure

| Flag                      | Default  | Description                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | Expose push endpoint via tailscale: `funnel`, `serve`, or `off`. |
| `--tailscale-path <path>` | (none)   | Path for tailscale serve/funnel.                                 |
| `--tailscale-target <t>`  | (none)   | Tailscale serve/funnel target (port, `host:port`, or URL).       |

### Output

| Flag     | Description                                       |
| -------- | ------------------------------------------------- |
| `--json` | Print a machine-readable summary instead of text. |

## `webhooks gmail run`

Run `gog watch serve` plus the watch auto-renew loop in the foreground.

```bash
openclaw webhooks gmail run --account you@example.com
```

`run` accepts the same `gog watch serve`, OpenClaw delivery, Pub/Sub, and Tailscale flags as `setup`, except:

- `--account` is **optional** on `run` (it falls back to the configured account).
- `run` does **not** accept `--project`, `--push-endpoint`, or `--json`.
- `run` flags have no built-in defaults; missing values fall back to the values written by `setup`.

| Category          | Flags                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`, `--topic`, `--subscription`, `--label`                              |
| OpenClaw delivery | `--hook-url`, `--hook-token`, `--push-token`                                     |
| `gog watch serve` | `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes` |
| Tailscale         | `--tailscale`, `--tailscale-path`, `--tailscale-target`                          |

> **Note:** For `run`, the `--topic` value is the full Pub/Sub topic path (`projects/.../topics/...`), not just the short topic name.


## End-to-end flow

See [Gmail Pub/Sub integration](/automation/cron-jobs#gmail-pubsub-integration) for the GCP project, OAuth, and gateway-side setup that pairs with these CLI commands.

## Related

- [CLI reference](/cli)
- [Webhook automation](/automation/webhook)
- [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration)


## openclaw wiki Command

# `openclaw wiki`

Inspect and maintain the `memory-wiki` vault.

Provided by the bundled `memory-wiki` plugin.

Related:

- [Memory Wiki plugin](/plugins/memory-wiki)
- [Memory Overview](/concepts/memory)
- [CLI: memory](/cli/memory)

## What it is for

Use `openclaw wiki` when you want a compiled knowledge vault with:

- wiki-native search and page reads
- provenance-rich syntheses
- contradiction and freshness reports
- bridge imports from the active memory plugin
- optional Obsidian CLI helpers

## Common commands

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "who should I ask about Teams?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Summary" \
  --body "Short synthesis body" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Still active?"

openclaw wiki bridge import
openclaw wiki unsafe-local import

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Commands

### `wiki status`

Inspect current vault mode, health, and Obsidian CLI availability.

Use this first when you are unsure whether the vault is initialized, bridge mode
is healthy, or Obsidian integration is available.

When bridge mode is active and configured to read memory artifacts, this command
queries the running Gateway so it sees the same active memory plugin context as
agent/runtime memory.

### `wiki doctor`

Run wiki health checks and surface configuration or vault problems.

When bridge mode is active and configured to read memory artifacts, this command
queries the running Gateway before building the report. Disabled bridge imports
and bridge configs that do not read memory artifacts remain local/offline.

Typical issues include:

- bridge mode enabled without public memory artifacts
- invalid or missing vault layout
- missing external Obsidian CLI when Obsidian mode is expected

### `wiki init`

Create the wiki vault layout and starter pages.

This initializes the root structure, including top-level indexes and cache
directories.

### `wiki ingest <path-or-url>`

Import content into the wiki source layer.

Notes:

- URL ingest is controlled by `ingest.allowUrlIngest`
- imported source pages keep provenance in frontmatter
- auto-compile can run after ingest when enabled

### `wiki compile`

Rebuild indexes, related blocks, dashboards, and compiled digests.

This writes stable machine-facing artifacts under:

- `.openclaw-wiki/cache/agent-digest.json`
- `.openclaw-wiki/cache/claims.jsonl`

If `render.createDashboards` is enabled, compile also refreshes report pages.

### `wiki lint`

Lint the vault and report:

- structural issues
- provenance gaps
- contradictions
- open questions
- low-confidence pages/claims
- stale pages/claims

Run this after meaningful wiki updates.

### `wiki search <query>`

Search wiki content.

Behavior depends on config:

- `search.backend`: `shared` or `local`
- `search.corpus`: `wiki`, `memory`, or `all`
- `--mode`: `auto`, `find-person`, `route-question`, `source-evidence`, or
  `raw-claim`

Use `wiki search` when you want wiki-specific ranking or provenance details.
For one broad shared recall pass, prefer `openclaw memory search` when the
active memory plugin exposes shared search.

Search modes help the agent choose the right surface:

- `find-person`: aliases, handles, socials, canonical IDs, and person pages
- `route-question`: ask-for/best-used-for hints and relationship context
- `source-evidence`: source pages and structured evidence fields
- `raw-claim`: structured claim text with claim/evidence metadata

Examples:

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "who knows Teams rollout?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "strong route Teams" --mode raw-claim --json
```

Text output includes `Claim:` and `Evidence:` lines when a result matches a
structured claim. JSON output additionally exposes `matchedClaimId`,
`matchedClaimStatus`, `matchedClaimConfidence`, `evidenceKinds`, and
`evidenceSourceIds` for agent-side drilldown.

### `wiki get <lookup>`

Read a wiki page by id or relative path.

Examples:

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

Apply narrow mutations without freeform page surgery.

Supported flows include:

- create/update a synthesis page
- update page metadata
- attach source ids
- add questions
- add contradictions
- update confidence/status
- write structured claims

This command exists so the wiki can evolve safely without manually editing
managed blocks.

### `wiki bridge import`

Import public memory artifacts from the active memory plugin into bridge-backed
source pages.

Use this in `bridge` mode when you want the latest exported memory artifacts
pulled into the wiki vault.

For active bridge artifact reads, the CLI routes the import through Gateway RPC
so the import uses the runtime memory plugin context. If bridge imports are
disabled or artifact reads are turned off, the command keeps the local/offline
zero-import behavior.

### `wiki unsafe-local import`

Import from explicitly configured local paths in `unsafe-local` mode.

This is intentionally experimental and same-machine only.

### `wiki obsidian ...`

Obsidian helper commands for vaults running in Obsidian-friendly mode.

Subcommands:

- `status`
- `search`
- `open`
- `command`
- `daily`

These require the official `obsidian` CLI on `PATH` when
`obsidian.useOfficialCli` is enabled.

## Practical usage guidance

- Use `wiki search` + `wiki get` when provenance and page identity matter.
- Use `wiki apply` instead of hand-editing managed generated sections.
- Use `wiki lint` before trusting contradictory or low-confidence content.
- Use `wiki compile` after bulk imports or source changes when you want fresh
  dashboards and compiled digests immediately.
- Use `wiki bridge import` when bridge mode depends on newly exported memory
  artifacts.

## Configuration tie-ins

`openclaw wiki` behavior is shaped by:

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

See [Memory Wiki plugin](/plugins/memory-wiki) for the full config model.

## Related

- [CLI reference](/cli)
- [Memory wiki](/plugins/memory-wiki)

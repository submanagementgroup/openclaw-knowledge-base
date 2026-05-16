---
domain: automation
topic: "Automation Hooks: Event-Driven Triggers"
type: reference
keywords:
  - hooks
  - automation hooks
  - event hooks
  - trigger
  - onMessage hook
  - onStart hook
source: automation/hooks.md
---

Hooks are small scripts that run when something happens inside the Gateway. They can be discovered from directories and inspected with `openclaw hooks`. The Gateway loads internal hooks only after you enable hooks or configure at least one hook entry, hook pack, legacy handler, or extra hook directory.

There are two kinds of hooks in OpenClaw:

- **Internal hooks** (this page): run inside the Gateway when agent events fire, like `/new`, `/reset`, `/stop`, or lifecycle events.
- **Webhooks**: external HTTP endpoints that let other systems trigger work in OpenClaw. See [Webhooks](/automation/cron-jobs#webhooks).

Hooks can also be bundled inside plugins. `openclaw hooks list` shows both standalone hooks and plugin-managed hooks.

## Quick start

```bash
# List available hooks
openclaw hooks list

# Enable a hook
openclaw hooks enable session-memory

# Check hook status
openclaw hooks check

# Get detailed information
openclaw hooks info session-memory
```

## Event types

| Event                    | When it fires                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | `/new` command issued                                      |
| `command:reset`          | `/reset` command issued                                    |
| `command:stop`           | `/stop` command issued                                     |
| `command`                | Any command event (general listener)                       |
| `session:compact:before` | Before compaction summarizes history                       |
| `session:compact:after`  | After compaction completes                                 |
| `session:patch`          | When session properties are modified                       |
| `agent:bootstrap`        | Before workspace bootstrap files are injected              |
| `gateway:startup`        | After channels start and hooks are loaded                  |
| `gateway:shutdown`       | When gateway shutdown begins                               |
| `gateway:pre-restart`    | Before an expected gateway restart                         |
| `message:received`       | Inbound message from any channel                           |
| `message:transcribed`    | After audio transcription completes                        |
| `message:preprocessed`   | After media and link preprocessing completes or is skipped |
| `message:sent`           | Outbound message delivered                                 |

## Writing hooks

### Hook structure

Each hook is a directory containing two files:

```
my-hook/
├── HOOK.md          # Metadata + documentation
└── handler.ts       # Handler implementation
```

### HOOK.md format

```markdown
---
name: my-hook
description: "Short description of what this hook does"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# My Hook

Detailed documentation goes here.
```

**Metadata fields** (`metadata.openclaw`):

| Field      | Description                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | Display emoji for CLI                                |
| `events`   | Array of events to listen for                        |
| `export`   | Named export to use (defaults to `"default"`)        |
| `os`       | Required platforms (e.g., `["darwin", "linux"]`)     |
| `requires` | Required `bins`, `anyBins`, `env`, or `config` paths |
| `always`   | Bypass eligibility checks (boolean)                  |
| `install`  | Installation methods                                 |

### Handler implementation

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] New command triggered`);
  // Your logic here

  // Optionally send message to user
  event.messages.push("Hook executed!");
};

export default handler;
```

Each event includes: `type`, `action`, `sessionKey`, `timestamp`, `messages` (push to send to user), and `context` (event-specific data). Agent and tool plugin hook contexts can also include `trace`, a read-only W3C-compatible diagnostic trace context that plugins may pass into structured logs for OTEL correlation.

### Event context highlights

**Command events** (`command:new`, `command:reset`): `context.sessionEntry`, `context.previousSessionEntry`, `context.commandSource`, `context.workspaceDir`, `context.cfg`.

**Message events** (`message:received`): `context.from`, `context.content`, `context.channelId`, `context.metadata` (provider-specific data including `senderId`, `senderName`, `guildId`). `context.content` prefers a nonblank command body for command-like messages, then falls back to the raw inbound body and generic body; it does not include agent-only enrichment such as thread history or link summaries.

**Message events** (`message:sent`): `context.to`, `context.content`, `context.success`, `context.channelId`.

**Message events** (`message:transcribed`): `context.transcript`, `context.from`, `context.channelId`, `context.mediaPath`.

**Message events** (`message:preprocessed`): `context.bodyForAgent` (final enriched body), `context.from`, `context.channelId`.

**Bootstrap events** (`agent:bootstrap`): `context.bootstrapFiles` (mutable array), `context.agentId`.

**Session patch events** (`session:patch`): `context.sessionEntry`, `context.patch` (only changed fields), `context.cfg`. Only privileged clients can trigger patch events.

**Compaction events**: `session:compact:before` includes `messageCount`, `tokenCount`. `session:compact:after` adds `compactedCount`, `summaryLength`, `tokensBefore`, `tokensAfter`.

`command:stop` observes the user issuing `/stop`; it is cancellation/command
lifecycle, not an agent-finalization gate. Plugins that need to inspect a
natural final answer and ask the agent for one more pass should use the typed
plugin hook `before_agent_finalize` instead. See [Plugin hooks](/plugins/hooks).

**Gateway lifecycle events**: `gateway:shutdown` includes `reason` and `restartExpectedMs` and fires when gateway shutdown begins. `gateway:pre-restart` includes the same context but only fires when shutdown is part of an expected restart and a finite `restartExpectedMs` value is supplied. During shutdown, each lifecycle hook wait is best-effort and bounded so shutdown continues if a handler stalls.

Between the `gateway:shutdown` (or `gateway:pre-restart`) event and the rest of the shutdown sequence, the gateway also fires a typed `session_end` plugin hook for every session that was still active when the process stopped. The event's `reason` is `shutdown` for a plain SIGTERM/SIGINT stop and `restart` when the close was scheduled as part of an expected restart. This drain is bounded so a slow `session_end` handler cannot block process exit, and sessions that have already been finalized through replace / reset / delete / compaction are skipped to avoid double-firing.

## Hook discovery

Hooks are discovered from these directories, in order of increasing override precedence:

1. **Bundled hooks**: shipped with OpenClaw
2. **Plugin hooks**: hooks bundled inside installed plugins
3. **Managed hooks**: `~/.openclaw/hooks/` (user-installed, shared across workspaces). Extra directories from `hooks.internal.load.extraDirs` share this precedence.
4. **Workspace hooks**: `<workspace>/hooks/` (per-agent, disabled by default until explicitly enabled)

Workspace hooks can add new hook names but cannot override bundled, managed, or plugin-provided hooks with the same name.

The Gateway skips internal hook discovery on startup until internal hooks are configured. Enable a bundled or managed hook with `openclaw hooks enable <name>`, install a hook pack, or set `hooks.internal.enabled=true` to opt in. When you enable one named hook, the Gateway loads only that hook's handler; `hooks.internal.enabled=true`, extra hook directories, and legacy handlers opt into broad discovery.

### Hook packs

Hook packs are npm packages that export hooks via `openclaw.hooks` in `package.json`. Install with:

```bash
openclaw plugins install <path-or-spec>
```

Npm specs are registry-only (package name + optional exact version or dist-tag). Git/URL/file specs and semver ranges are rejected.

## Bundled hooks

| Hook                  | Events                                            | What it does                                                   |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| session-memory        | `command:new`, `command:reset`                    | Saves session context to `<workspace>/memory/`                 |
| bootstrap-extra-files | `agent:bootstrap`                                 | Injects additional bootstrap files from glob patterns          |
| command-logger        | `command`                                         | Logs all commands to `~/.openclaw/logs/commands.log`           |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | Sends visible chat notices when session compaction starts/ends |
| boot-md               | `gateway:startup`                                 | Runs `BOOT.md` when the gateway starts                         |

Enable any bundled hook:

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### session-memory details

Extracts the last 15 user/assistant messages and saves to `<workspace>/memory/YYYY-MM-DD-HHMM.md` using the host local date. Memory capture runs in the background so `/new` and `/reset` acknowledgements are not delayed by transcript reads or optional slug generation. Set `hooks.internal.entries.session-memory.llmSlug: true` to generate descriptive filename slugs with the configured model. Requires `workspace.dir` to be configured.

<a id="bootstrap-extra-files"></a>

### bootstrap-extra-f
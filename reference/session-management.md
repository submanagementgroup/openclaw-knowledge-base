---
domain: reference
topic: "Session Management and Compaction Reference: Internals and Configuration"
type: reference
keywords:
  - session management
  - compaction reference
  - session internals
  - transcript management
related:
  - concepts/session
  - concepts/compaction
source: reference/session-management-compaction.md
---

Session management and compaction internals reference.

OpenClaw manages sessions end-to-end across these areas:

- **Session routing** (how inbound messages map to a `sessionKey`)
- **Session store** (`sessions.json`) and what it tracks
- **Transcript persistence** (`*.jsonl`) and its structure
- **Transcript hygiene** (provider-specific fixups before runs)
- **Context limits** (context window vs tracked tokens)
- **Compaction** (manual and auto-compaction) and where to hook pre-compaction work
- **Silent housekeeping** (memory writes that should not produce user-visible output)

If you want a higher-level overview first, start with:

- [Session management](/concepts/session)
- [Compaction](/concepts/compaction)
- [Memory overview](/concepts/memory)
- [Memory search](/concepts/memory-search)
- [Session pruning](/concepts/session-pruning)
- [Transcript hygiene](/reference/transcript-hygiene)

---

## Source of truth: the Gateway

OpenClaw is designed around a single **Gateway process** that owns session state.

- UIs (macOS app, web Control UI, TUI) should query the Gateway for session lists and token counts.
- In remote mode, session files are on the remote host; “checking your local Mac files” won’t reflect what the Gateway is using.

---

## Two persistence layers

OpenClaw persists sessions in two layers:

1. **Session store (`sessions.json`)**
   - Key/value map: `sessionKey -> SessionEntry`
   - Small, mutable, safe to edit (or delete entries)
   - Tracks session metadata (current session id, last activity, toggles, token counters, etc.)

2. **Transcript (`<sessionId>.jsonl`)**
   - Append-only transcript with tree structure (entries have `id` + `parentId`)
   - Stores the actual conversation + tool calls + compaction summaries
   - Used to rebuild the model context for future turns
   - Large pre-compaction debug checkpoints are skipped once the active
     transcript exceeds the checkpoint size cap, avoiding a second giant
     `.checkpoint.*.jsonl` copy.

---

## On-disk locations

Per agent, on the Gateway host:

- Store: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcripts: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Telegram topic sessions: `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw resolves these via `src/config/sessions.ts`.

---

## Store maintenance and disk controls

Session persistence has automatic maintenance controls (`session.maintenance`) for `sessions.json`, transcript artifacts, and trajectory sidecars:

- `mode`: `warn` (default) or `enforce`
- `pruneAfter`: stale-entry age cutoff (default `30d`)
- `maxEntries`: cap entries in `sessions.json` (default `500`)
- `resetArchiveRetention`: retention for `*.reset.<timestamp>` transcript archives (default: same as `pruneAfter`; `false` disables cleanup)
- `maxDiskBytes`: optional sessions-directory budget
- `highWaterBytes`: optional target after cleanup (default `80%` of `maxDiskBytes`)

Normal Gateway writes batch `maxEntries` cleanup for production-sized caps, so a store may briefly exceed the configured cap before the next high-water cleanup rewrites it back down. `openclaw sessions cleanup --enforce` still applies the configured cap immediately.

OpenClaw no longer creates automatic `sessions.json.bak.*` rotation backups during Gateway writes. The legacy `session.maintenance.rotateBytes` key is ignored and `openclaw doctor --fix` removes it from older configs.

Enforcement order for disk budget cleanup (`mode: "enforce"`):

1. Remove oldest archived, orphan transcript, or orphan trajectory artifacts first.
2. If still above the target, evict oldest session entries and their transcript/trajectory files.
3. Keep going until usage is at or below `highWaterBytes`.

In `mode: "warn"`, OpenClaw reports potential evictions but does not mutate the store/files.

Run maintenance on demand:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Cron sessions and run logs

Isolated cron runs also create session entries/transcripts, and they have dedicated retention controls:

- `cron.sessionRetention` (default `24h`) prunes old isolated cron run sessions from the session store (`false` disables).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` prune `~/.openclaw/cron/runs/<jobId>.jsonl` files (defaults: `2_000_000` bytes and `2000` lines).

When cron force-creates a new isolated run session, it sanitizes the previous
`cron:<jobId>` session entry before writing the new row. It carries safe
preferences such as thinking/fast/verbose settings, labels, and explicit
user-selected model/auth overrides. It drops ambient conversation context such
as channel/group routing, send or queue policy, elevation, origin, and ACP
runtime binding so a fresh isolated run cannot inherit stale delivery or
runtime authority from an older run.

---

## Session keys (`sessionKey`)

A `sessionKey` identifies _which conversation bucket_ you’re in (routing + isolation).

Common patterns:

- Main/direct chat (per agent): `agent:<agentId>:<mainKey>` (default `main`)
- Group: `agent:<agentId>:<channel>:group:<id>`
- Room/channel (Discord/Slack): `agent:<agentId>:<channel>:channel:<id>` or `...:room:<id>`
- Cron: `cron:<job.id>`
- Webhook: `hook:<uuid>` (unless overridden)

The canonical rules are documented at [/concepts/session](/concepts/session).

---

## Session ids (`sessionId`)

Each `sessionKey` points at a current `sessionId` (the transcript file that continues the conversation).

Rules of thumb:

- **Reset** (`/new`, `/reset`) creates a new `sessionId` for that `sessionKey`.
- **Daily reset** (default 4:00 AM local time on the gateway host) creates a new `sessionId` on the next message after the reset boundary.
- **Idle expiry** (`session.reset.idleMinutes` or legacy `session.idleMinutes`) creates a new `sessionId` when a message arrives after the idle window. When daily + idle are both configured, whichever expires first wins.
- **System events** (heartbeat, cron wakeups, exec notifications, gateway bookkeeping) may mutate the session row but do not extend daily/idle reset freshness. Reset rollover discards queued system-event notices for the previous session before the fresh prompt is built.
- **Thread parent fork guard** (`session.parentForkMaxTokens`, default `100000`) skips parent transcript forking when the parent session is already too large; the new thread starts fresh. Set `0` to disable.

Implementation detail: the decision happens in `initSessionState()` in `src/auto-reply/reply/session.ts`.

---

## Session store schema (`sessions.json`)

The store’s value type is `SessionEntry` in `src/config/sessions.ts`.

Key fields (not exhaustive):

- `sessionId`: current transcript id (filename is derived from this unless `sessionFile` is set)
- `sessionStartedAt`: start timestamp for the current `sessionId`; daily reset
  freshness uses this. Legacy rows may derive it from the JSONL session header.
- `lastInteractionAt`: last real user/channel interaction timestamp; idle reset
  freshness uses this so heartbeat, cron, and exec events do not keep sessions
  alive. Legacy rows without this field fall back to the recovered session start
  time for idle freshness.
- `updatedAt`: last store-row mutation timestamp, used for listing, pruning, and
  bookkeeping. It is not the authority for daily/idle reset freshness.
- `sessionFile`: optional explicit transcript path override
- `chatType`: `direct | group | room` (helps UIs and send policy)
- `provider`, `subject`, `room`, `space`, `displayName`: metadata for group/channel labeling
- Toggles:
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (per-session override)
- Model selection:
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Token counters (best-effort / provider-dependent):
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: how often auto-compaction completed for this session key
- `memoryFlushAt`: timestamp for the last pre-compaction memory

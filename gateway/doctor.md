---
domain: gateway
topic: "openclaw doctor: Diagnostics and Auto-Fix"
type: procedure
keywords:
  - openclaw doctor
  - doctor --fix
  - doctor --yes
  - diagnostics
  - config repair
  - last-known-good
  - auth repair
source: gateway/doctor.md
---

`openclaw doctor` is the repair + migration tool for OpenClaw. It fixes stale config/state, checks health, and provides actionable repair steps.

## Quick start

```bash
openclaw doctor
```

### Headless and automation modes

**--yes:**

```bash
    openclaw doctor --yes
    ```

    Accept defaults without prompting (including restart/service/sandbox repair steps when applicable).

  
**--repair:**

```bash
    openclaw doctor --repair
    ```

    Apply recommended repairs without prompting (repairs + restarts where safe).

  
**--repair --force:**

```bash
    openclaw doctor --repair --force
    ```

    Apply aggressive repairs too (overwrites custom supervisor configs).

  
**--non-interactive:**

```bash
    openclaw doctor --non-interactive
    ```

    Run without prompts and only apply safe migrations (config normalization + on-disk state moves). Skips restart/service/sandbox actions that require human confirmation. Legacy state migrations run automatically when detected.

  
**--deep:**

```bash
    openclaw doctor --deep
    ```

    Scan system services for extra gateway installs (launchd/systemd/schtasks).

  
If you want to review changes before writing, open the config file first:

```bash
cat ~/.openclaw/openclaw.json
```

## What it does (summary)

### Health, UI, and updates

- Optional pre-flight update for git installs (interactive only).
    - UI protocol freshness check (rebuilds Control UI when the protocol schema is newer).
    - Health check + restart prompt.
    - Skills status summary (eligible/missing/blocked) and plugin status.

  ### Config and migrations

- Config normalization for legacy values.
    - Talk config migration from legacy flat `talk.*` fields into `talk.provider` + `talk.providers.<provider>`.
    - Browser migration checks for legacy Chrome extension configs and Chrome MCP readiness.
    - OpenCode provider override warnings (`models.providers.opencode` / `models.providers.opencode-go`).
    - Codex OAuth shadowing warnings (`models.providers.openai-codex`).
    - OAuth TLS prerequisites check for OpenAI Codex OAuth profiles.
    - Plugin/tool allowlist warnings when `plugins.allow` is restrictive but tool policy still asks for wildcard or plugin-owned tools.
    - Legacy on-disk state migration (sessions/agent dir/WhatsApp auth).
    - Legacy plugin manifest contract key migration (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
    - Legacy cron store migration (`jobId`, `schedule.cron`, top-level delivery/payload fields, payload `provider`, simple `notify: true` webhook fallback jobs).
    - Legacy whole-agent runtime-policy cleanup; provider/model runtime policy is the active route selector.
    - Stale plugin config cleanup when plugins are enabled; when `plugins.enabled=false`, stale plugin references are treated as inert containment config and are preserved.

  ### State and integrity

- Session lock file inspection and stale lock cleanup.
    - Session transcript repair for duplicated prompt-rewrite branches created by affected 2026.4.24 builds.
    - Wedged subagent restart-recovery tombstone detection, with `--fix` support for clearing stale aborted recovery flags so startup does not keep treating the child as restart-aborted.
    - State integrity and permissions checks (sessions, transcripts, state dir).
    - Config file permission checks (chmod 600) when running locally.
    - Model auth health: checks OAuth expiry, can refresh expiring tokens, and reports auth-profile cooldown/disabled states.
    - Extra workspace dir detection (`~/openclaw`).

  ### Gateway, services, and supervisors

- Sandbox image repair when sandboxing is enabled.
    - Legacy service migration and extra gateway detection.
    - Matrix channel legacy state migration (in `--fix` / `--repair` mode).
    - Gateway runtime checks (service installed but not running; cached launchd label).
    - Channel status warnings (probed from the running gateway).
    - Channel-specific permission checks live under `openclaw channels capabilities`; for example, Discord voice channel permissions are audited with `openclaw channels capabilities --channel discord --target channel:<channel-id>`.
    - WhatsApp responsiveness checks for degraded Gateway event-loop health with local TUI clients still running; `--fix` stops only verified local TUI clients.
    - Codex route repair for legacy `openai-codex/*` model refs in primary models, fallbacks, heartbeat/subagent/compaction overrides, hooks, channel model overrides, and session route pins; `--fix` rewrites them to `openai/*`, removes stale session/whole-agent runtime pins, and leaves canonical OpenAI agent refs on the default Codex harness.
    - Supervisor config audit (launchd/systemd/schtasks) with optional repair.
    - Embedded proxy environment cleanup for gateway services that captured shell `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` values during install or update.
    - Gateway runtime best-practice checks (Node vs Bun, version-manager paths).
    - Gateway port collision diagnostics (default `18789`).

  ### Auth, security, and pairing

- Security warnings for open DM policies.
    - Gateway auth checks for local token mode (offers token generation when no token source exists; does not overwrite token SecretRef configs).
    - Device pairing trouble detection (pending first-time pair requests, pending role/scope upgrades, stale local device-token cache drift, and paired-record auth drift).

  ### Workspace and shell

- systemd linger check on Linux.
    - Workspace bootstrap file size check (truncation/near-limit warnings for context files).
    - Skills readiness check for the default agent; reports allowed skills with missing bins, env, config, or OS requirements, and `--fix` can disable unavailable skills in `skills.entries`.
    - Shell completion status check and auto-install/upgrade.
    - Memory search embedding provider readiness check (local model, remote API key, or QMD binary).
    - Source install checks (pnpm workspace mismatch, missing UI assets, missing tsx binary).
    - Writes updated config + wizard metadata.

  ## Dreams UI backfill and reset

The Control UI Dreams scene includes **Backfill**, **Reset**, and **Clear Grounded** actions for the grounded dreaming workflow. These actions use gateway doctor-style RPC methods, but they are **not** part of `openclaw doctor` CLI repair/migration.

What they do:

- **Backfill** scans historical `memory/YYYY-MM-DD.md` files in the active workspace, runs the grounded REM diary pass, and writes reversible backfill entries into `DREAMS.md`.
- **Reset** removes only those marked backfill diary entries from `DREAMS.md`.
- **Clear Grounded** removes only staged grounded-only short-term entries that came from historical replay and have not accumulated live recall or daily support yet.

What they do **not** do by themselves:

- they do not edit `MEMORY.md`
- they do not run full doctor migrations
- they do not automatically stage grounded candidates into the live short-term promotion store unless you explicitly run the staged CLI path first

If you want grounded historical replay to influence the normal deep promotion lane, use the CLI flow instead:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

That stages grounded durable candidates into the short-term dreaming store while keeping `DREAMS.md` as the review surface.

## Detailed behavior and rationale

### 0. Optional update (git installs)

If this is a git checkout and doctor is running interactively, it offers to update (fetch/rebase/build) before running doctor.
  ### 1. Config normalization

If the config contains legacy value shapes (for example `messages.ackReaction` without a channel-specific override), doctor normalizes them into the current schema.

    That includes legacy Talk flat fields. Current public Talk speech config is `talk.provider` + `talk.providers.<provider>`, and realtime voice config is `talk.realtime.*`. Doctor rewrites old `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` shapes into the provider map, and rewrites legacy top-level realtime selectors (`talk.mode`, `talk.transport`, `talk.brain`, `talk.model`, `talk.voice`) into `talk.realtime`.

    Doctor also warns when `plugins.allow` is non-empty and tool policy uses
    wildcard or plugin-owned tool entries. `tools.allow: ["*"]` only matches tools
    from plugins that actually load; it does not bypass the exclusive plugin
    allowlist. Doctor writes `plugins.bundledDiscovery: "compat"` for migrated
    legacy allowlist configs to preserve existing bundled provider behavior, and
    then points to the stricter `"allowlist"` setting.

  ### 2. Legacy config key migrations

When the config contains deprecated keys, other commands refuse to run and ask you to run `openclaw doctor`.

    Doctor will:

    - Explain which legacy keys were found.
    - Show the migration it applied.
    - Rewrite `~/.openclaw/openclaw.json` with the updated schema.

    Gateway startup refuses legacy config formats and asks you to run `openclaw doctor --fix`; it does not rewrite `openclaw.json` on startup. Cron job store migrations are also handled by `openclaw doctor --fix`.

    Current migrations:

    - `routing.allowFrom` → `channels.whatsapp.allowFrom`
    - `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
    - `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
    - `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
    - `channels.telegram.requireMention` → `channels.telegram.groups."*".requireMention`
    - configured-channel configs missing visible reply policy → `messages.groupChat.visibleReplies: "message_tool"`
    - `routing.queue` → `messages.queue`
    - `routing.bindings` → top-level `bindings`
    - `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
    - legacy `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
    - legacy top-level realtime Talk selectors (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) + `talk.provider`/`talk.providers` → `talk.realtime`
    - `routing.agentToAgent` → `tools.agentToAgent`
    - `routing.transcribeAudio` → `tools.media.audio.models`
    - `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
    - `messages.tts.provider: "edge"` and `messages.tts.providers.edge` → `messages.tts.provider: "microsoft"` and `messages.tts.providers.microsoft`
    - `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
    - `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
    - `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
    - `plugins.entries.voice-call.config.tts.provider: "edge"` and `plugins.entries.voice-call.config.tts.providers.edge` → `provider: "microsoft"` and `providers.microsoft`
    - `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
    - `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
    - `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
    - `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold` → `plugins.entries.voice-call.config.streaming.providers.openai.*`
    - `bindings[].match.accountI
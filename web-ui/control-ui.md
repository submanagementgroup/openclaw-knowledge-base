---
domain: web-ui
topic: "Web UI: Control Ui"
type: procedure
keywords:
  - control-ui
  - web UI
  - control UI
  - web interface
source: web/control-ui.md
---

The Control UI is a small **Vite + Lit** single-page app served by the Gateway:

- default: `http://<host>:18789/`
- optional prefix: set `gateway.controlUi.basePath` (e.g. `/openclaw`)

It speaks **directly to the Gateway WebSocket** on the same port.

## Quick open (local)

If the Gateway is running on the same computer, open:

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (or [http://localhost:18789/](http://localhost:18789/))

If the page fails to load, start the Gateway first: `openclaw gateway`.

Auth is supplied during the WebSocket handshake via:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Tailscale Serve identity headers when `gateway.auth.allowTailscale: true`
- trusted-proxy identity headers when `gateway.auth.mode: "trusted-proxy"`

The dashboard settings panel keeps a token for the current browser tab session and selected gateway URL; passwords are not persisted. Onboarding usually generates a gateway token for shared-secret auth on first connect, but password auth works too when `gateway.auth.mode` is `"password"`.

## Device pairing (first connection)

When you connect to the Control UI from a new browser or device, the Gateway usually requires a **one-time pairing approval**. This is a security measure to prevent unauthorized access.

**What you'll see:** "disconnected (1008): pairing required"

**List pending requests**

```bash
    openclaw devices list
    ```
  
**Approve by request ID**

```bash
    openclaw devices approve <requestId>
    ```
  
If the browser retries pairing with changed auth details (role/scopes/public key), the previous pending request is superseded and a new `requestId` is created. Re-run `openclaw devices list` before approval.

If the browser is already paired and you change it from read access to write/admin access, this is treated as an approval upgrade, not a silent reconnect. OpenClaw keeps the old approval active, blocks the broader reconnect, and asks you to approve the new scope set explicitly.

Once approved, the device is remembered and won't require re-approval unless you revoke it with `openclaw devices revoke --device <id> --role <role>`. See [Devices CLI](/cli/devices) for token rotation and revocation.

> **Note:** - Direct local loopback browser connections (`127.0.0.1` / `localhost`) are auto-approved.
- Tailscale Serve can skip the pairing round trip for Control UI operator sessions when `gateway.auth.allowTailscale: true`, Tailscale identity verifies, and the browser presents its device identity.
- Direct Tailnet binds, LAN browser connects, and browser profiles without device identity still require explicit approval.
- Each browser profile generates a unique device ID, so switching browsers or clearing browser data will require re-pairing.



## Personal identity (browser-local)

The Control UI supports a per-browser personal identity (display name and avatar) attached to outgoing messages for attribution in shared sessions. It lives in browser storage, is scoped to the current browser profile, and is not synced to other devices or persisted server-side beyond the normal transcript authorship metadata on messages you actually send. Clearing site data or switching browsers resets it to empty.

The same browser-local pattern applies to the assistant avatar override. Uploaded assistant avatars overlay the gateway-resolved identity on the local browser only and never round-trip through `config.patch`. The shared `ui.assistant.avatar` config field is still available for non-UI clients writing the field directly (such as scripted gateways or custom dashboards).

## Runtime config endpoint

The Control UI fetches its runtime settings from `/__openclaw/control-ui-config.json`. That endpoint is gated by the same gateway auth as the rest of the HTTP surface: unauthenticated browsers cannot fetch it, and a successful fetch requires either an already valid gateway token/password, Tailscale Serve identity, or a trusted-proxy identity.

## Language support

The Control UI can localize itself on first load based on your browser locale. To override it later, open **Overview -> Gateway Access -> Language**. The locale picker lives in the Gateway Access card, not under Appearance.

- Supported locales: `en`, `zh-CN`, `zh-TW`, `pt-BR`, `de`, `es`, `ja-JP`, `ko`, `fr`, `ar`, `it`, `tr`, `uk`, `id`, `pl`, `th`, `vi`, `nl`, `fa`
- Non-English translations are lazy-loaded in the browser.
- The selected locale is saved in browser storage and reused on future visits.
- Missing translation keys fall back to English.

Docs translations are generated for the same non-English locale set, but the docs site's built-in Mintlify language picker is limited to the locale codes Mintlify accepts. Thai (`th`) and Persian (`fa`) docs are still generated in the publish repo; they may not appear in that picker until Mintlify supports those codes.

## Appearance themes

The Appearance panel keeps the built-in Claw, Knot, and Dash themes, plus one browser-local tweakcn import slot. To import a theme, open [tweakcn editor](https://tweakcn.com/editor/theme), choose or create a theme, click **Share**, and paste the copied theme link into Appearance. The importer also accepts `https://tweakcn.com/r/themes/<id>` registry URLs, editor URLs like `https://tweakcn.com/editor/theme?theme=amethyst-haze`, relative `/themes/<id>` paths, raw theme IDs, and default theme names such as `amethyst-haze`.

Imported themes are stored only in the current browser profile. They are not written to gateway config and do not sync across devices. Replacing the imported theme updates the one local slot; clearing it switches the active theme back to Claw if the imported theme was selected.

## What it can do (today)

### Chat and Talk

- Chat with the model via Gateway WS (`chat.history`, `chat.send`, `chat.abort`, `chat.inject`).
    - Chat history refreshes request a bounded recent window with per-message text caps so large sessions do not force the browser to render a full transcript payload before the chat becomes usable.
    - Talk through browser realtime sessions. OpenAI uses direct WebRTC, Google Live uses a constrained one-use browser token over WebSocket, and backend-only realtime voice plugins use the Gateway relay transport. Client-owned provider sessions start with `talk.client.create`; Gateway relay sessions start with `talk.session.create`. The relay keeps provider credentials on the Gateway while the browser streams microphone PCM through `talk.session.appendAudio` and forwards `openclaw_agent_consult` provider tool calls through `talk.client.toolCall` for Gateway policy and the larger configured OpenClaw model.
    - Stream tool calls + live tool output cards in Chat (agent events).

  ### Channels, instances, sessions, dreams

- Channels: built-in plus bundled/external plugin channels status, QR login, and per-channel config (`channels.status`, `web.login.*`, `config.patch`).
    - Channel probe refreshes keep the previous snapshot visible while slow provider checks finish, and partial snapshots are labeled when a probe or audit exceeds its UI budget.
    - Instances: presence list + refresh (`system-presence`).
    - Sessions: list configured-agent sessions by default, fall back from stale unconfigured agent session keys, and apply per-session model/thinking/fast/verbose/trace/reasoning overrides (`sessions.list`, `sessions.patch`).
    - Dreams: dreaming status, enable/disable toggle, and Dream Diary reader (`doctor.memory.status`, `doctor.memory.dreamDiary`, `config.patch`).

  ### Cron, skills, nodes, exec approvals

- Cron jobs: list/add/edit/run/enable/disable + run history (`cron.*`).
    - Skills: status, enable/disable, install, API key updates (`skills.*`).
    - Nodes: list + caps (`node.list`).
    - Exec approvals: edit gateway or node allowlists + ask policy for `exec host=gateway/node` (`exec.approvals.*`).

  ### Config

- View/edit `~/.openclaw/openclaw.json` (`config.get`, `config.set`).
    - Apply + restart with validation (`config.apply`) and wake the last active session.
    - Writes include a base-hash guard to prevent clobbering concurrent edits.
    - Writes (`config.set`/`config.apply`/`config.patch`) preflight active SecretRef resolution for refs in the submitted config payload; unresolved active submitted refs are rejected before write.
    - Schema + form rendering (`config.schema` / `config.schema.lookup`, including field `title` / `description`, matched UI hints, immediate child summaries, docs metadata on nested object/wildcard/array/composition nodes, plus plugin + channel schemas when available); Raw JSON editor is available only when the snapshot has a safe raw round-trip.
    - If a snapshot cannot safely round-trip raw text, Control UI forces Form mode and disables Raw mode for that snapshot.
    - Raw JSON editor "Reset to saved" preserves the raw-authored shape (formatting, comments, `$include` layout) instead of re-rendering a flattened snapshot, so external edits survive a reset when the snapshot can safely round-trip.
    - Structured SecretRef object values are rendered read-only in form text inputs to prevent accidental object-to-string corruption.

  ### Debug, logs, update

- Debug: status/health/models snapshots + event log + manual RPC calls (`status`, `health`, `models.list`).
    - The event log includes Control UI refresh/RPC timings, slow chat/config render timings, and browser responsiveness entries for long animation frames or long tasks when the browser exposes those PerformanceObserver entry types.
    - Logs: live tail of gateway file logs with filter/export (`logs.tail`).
    - Update: run a package/git update + restart (`update.run`) with a restart report, then poll `update.status` after reconnect to verify the running gateway version.

  ### Cron jobs panel notes

- For isolated jobs, delivery defaults to announce summary. You can switch to none if you want internal-only runs.
    - Channel/target fields appear 
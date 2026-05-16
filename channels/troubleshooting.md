---
domain: channels
topic: "Channel Troubleshooting Guide"
type: integration
keywords:
  - channel troubleshooting
  - channel errors
  - connection issues
source: channels/troubleshooting.md
---

Use this page when a channel connects but behavior is wrong.

## Command ladder

Run these in order first:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Healthy baseline:

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`, `write-capable`, or `admin-capable`
- Channel probe shows transport connected and, where supported, `works` or `audit ok`

## WhatsApp

### WhatsApp failure signatures

| Symptom                             | Fastest check                                       | Fix                                                                                                                              |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Connected but no DM replies         | `openclaw pairing list whatsapp`                    | Approve sender or switch DM policy/allowlist.                                                                                    |
| Group messages ignored              | Check `requireMention` + mention patterns in config | Mention the bot or relax mention policy for that group.                                                                          |
| QR login times out with 408         | Check gateway `HTTPS_PROXY` / `HTTP_PROXY` env      | Set a reachable proxy; use `NO_PROXY` only for bypasses.                                                                         |
| Random disconnect/relogin loops     | `openclaw channels status --probe` + logs           | Recent reconnects are flagged even when currently connected; watch logs, restart the gateway, then relink if flapping continues. |
| Replies arrive seconds/minutes late | `openclaw doctor --fix`                             | Doctor stops verified stale local TUI clients when they are degrading the Gateway event loop.                                    |

Full troubleshooting: [WhatsApp troubleshooting](/channels/whatsapp#troubleshooting)

## Telegram

### Telegram failure signatures

| Symptom                              | Fastest check                                    | Fix                                                                                                                        |
| ------------------------------------ | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `/start` but no usable reply flow    | `openclaw pairing list telegram`                 | Approve pairing or change DM policy.                                                                                       |
| Bot online but group stays silent    | Verify mention requirement and bot privacy mode  | Disable privacy mode for group visibility or mention bot.                                                                  |
| Send failures with network errors    | Inspect logs for Telegram API call failures      | Fix DNS/IPv6/proxy routing to `api.telegram.org`.                                                                          |
| Startup reports `getMe returned 401` | Check configured token source                    | Re-copy or regenerate the BotFather token and update `botToken`, `tokenFile`, or default-account `TELEGRAM_BOT_TOKEN`.     |
| Polling stalls or reconnects slowly  | `openclaw logs --follow` for polling diagnostics | Upgrade; if restarts are false positives, tune `pollingStallThresholdMs`. Persistent stalls still point to proxy/DNS/IPv6. |
| `setMyCommands` rejected at startup  | Inspect logs for `BOT_COMMANDS_TOO_MUCH`         | Reduce plugin/skill/custom Telegram commands or disable native menus.                                                      |
| Upgraded and allowlist blocks you    | `openclaw security audit` and config allowlists  | Run `openclaw doctor --fix` or replace `@username` with numeric sender IDs.                                                |

Full troubleshooting: [Telegram troubleshooting](/channels/telegram#troubleshooting)

## Discord

### Discord failure signatures

| Symptom                                   | Fastest check                                                          | Fix                                                                                                                                                                     |
| ----------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bot online but no guild replies           | `openclaw channels status --probe`                                     | Allow guild/channel and verify message content intent.                                                                                                                  |
| Group messages ignored                    | Check logs for mention gating drops                                    | Mention bot or set guild/channel `requireMention: false`.                                                                                                               |
| Typing/token usage but no Discord message | Session log shows assistant text with `didSendViaMessagingTool: false` | The model answered privately instead of calling the message tool. Use a tool-call-reliable model, or set `messages.groupChat.visibleReplies: "automatic"` to auto-post. |
| DM replies missing                        | `openclaw pairing list discord`                                        | Approve DM pairing or adjust DM policy.                                                                                                                                 |

Full troubleshooting: [Discord troubleshooting](/channels/discord#troubleshooting)

## Slack

### Slack failure signatures

| Symptom                                | Fastest check                             | Fix                                                                                                                                                  |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Socket mode connected but no responses | `openclaw channels status --probe`        | Verify app token + bot token and required scopes; watch for `botTokenStatus` / `appTokenStatus = configured_unavailable` on SecretRef-backed setups. |
| DMs blocked                            | `openclaw pairing list slack`             | Approve pairing or relax DM policy.                                                                                                                  |
| Channel message ignored                | Check `groupPolicy` and channel allowlist | Allow the channel or switch policy to `open`.                                                                                                        |

Full troubleshooting: [Slack troubleshooting](/channels/slack#troubleshooting)

## iMessage

### iMessage failure signatures

| Symptom                              | Fastest check                                           | Fix                                                                   |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| `imsg` missing or fails on non-macOS | `openclaw channels status --probe --channel imessage`   | Run OpenClaw on the Messages Mac or use an SSH wrapper for `cliPath`. |
| Can send but no receive on macOS     | Check macOS privacy permissions for Messages automation | Re-grant TCC permissions and restart channel process.                 |
| DM sender blocked                    | `openclaw pairing list imessage`                        | Approve pairing or update allowlist.                                  |

Full troubleshooting:

- [iMessage troubleshooting](/channels/imessage#troubleshooting)

## Signal

### Signal failure signatures

| Symptom                         | Fastest check                              | Fix                                                      |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| Daemon reachable but bot silent | `openclaw channels status --probe`         | Verify `signal-cli` daemon URL/account and receive mode. |
| DM blocked                      | `openclaw pairing list signal`             | Approve sender or adjust DM policy.                      |
| Group replies do not trigger    | Check group allowlist and mention patterns | Add sender/group or loosen gating.                       |

Full troubleshooting: [Signal troubleshooting](/channels/signal#troubleshooting)

## QQ Bot

### QQ Bot failure signatures

| Symptom                         | Fastest check                               | Fix                                                             |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Bot replies "gone to Mars"      | Verify `appId` and `clientSecret` in config | Set credentials or restart the gateway.                         |
| No inbound messages             | `openclaw channels status --probe`          | Verify credentials on the QQ Open Platform.                     |
| Voice not transcribed           | Check STT provider config                   | Configure `channels.qqbot.stt` or `tools.media.aud
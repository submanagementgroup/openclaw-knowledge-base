---
domain: cli
topic: "openclaw directory: Channel Directory Lookups for Contacts, Groups, and Self"
type: reference
keywords:
  - openclaw directory
  - directory lookup
  - contacts list
  - groups list
  - peers list
  - channel directory
  - WhatsApp contacts
  - Telegram contacts
  - Slack users
  - Zalo directory
source: cli/directory.md
related:
  - cli/cli-overview
  - channels/channels-overview
---

`openclaw directory` performs directory lookups for channels that support it — contacts/peers, groups, and "me". Results are useful for finding IDs to pass to `openclaw message send --target ...`.

## Common Flags

- `--channel <name>`: channel id/alias (required when multiple channels are configured)
- `--account <id>`: account id (default: channel default)
- `--json`: output JSON for scripting

## Examples

```bash
# Look up peers on Slack
openclaw directory peers list --channel slack --query "U0"

# Use result in message send
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"

# List groups on Zalo
openclaw directory groups list --channel zalouser

# Get group members
openclaw directory groups members --channel zalouser --group-id <id>

# Get self ("me")
openclaw directory self --channel zalouser
```

## Channel-Specific ID Formats

| Channel | DM format | Group format |
|---------|-----------|--------------|
| WhatsApp | `+15551234567` | `1234567890-1234567890@g.us` |
| Telegram | `@username` or numeric chat id | numeric id |
| Slack | `user:U…` | `channel:C…` |
| Discord | `user:<id>` | `channel:<id>` |
| Matrix | `user:@user:server` | `room:!roomId:server` or `#alias:server` |
| MS Teams | `user:<id>` | `conversation:<id>` |
| Zalo Personal | thread id from `me`, `friend list`, `group list` | — |

## Subcommands

### `self`

Returns the "me" identity for the channel.

```bash
openclaw directory self --channel zalouser
```

### `peers list`

Lists contacts/users for the channel.

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

### `groups list` / `groups members`

Lists groups and their members.

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```

## Notes

- For many channels, results are config-backed (allowlists / configured groups) rather than a live provider directory.
- Default output is tab-separated `id` (and sometimes `name`); use `--json` for scripting.

## Related

- [CLI overview](/cli)
- [Channels overview](/channels/channels-overview)

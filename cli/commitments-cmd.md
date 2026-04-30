---
domain: cli
topic: "openclaw commitments: List and Dismiss Inferred Follow-Up Commitments"
type: reference
keywords:
  - openclaw commitments
  - inferred commitments
  - dismiss commitments
  - follow-up commitments
  - pending commitments
  - heartbeat commitments
  - commitment status
source: cli/commitments.md
related:
  - cli/cli-overview
  - concepts/commitments
  - automation/cron-jobs
---

`openclaw commitments` lists and manages inferred follow-up commitments — opt-in, short-lived memories created from conversation context. With no subcommand, it lists pending commitments.

## Usage

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## Options

- `--all`: show all statuses instead of only pending commitments
- `--agent <id>`: filter to one agent id
- `--status <status>`: filter by status — `pending`, `sent`, `dismissed`, `snoozed`, or `expired`
- `--json`: machine-readable JSON output

## Examples

```bash
# List pending commitments (default)
openclaw commitments

# List every stored commitment
openclaw commitments --all

# Filter to one agent
openclaw commitments --agent main

# Find snoozed commitments
openclaw commitments --status snoozed

# Dismiss one or more commitments
openclaw commitments dismiss cm_abc123 cm_def456

# Export as JSON
openclaw commitments --all --json
```

## Output Fields

Text output includes: commitment id, status, kind, earliest due time, scope, and suggested check-in text. JSON output also includes the commitment store path and full stored records.

## Related

- [Inferred commitments](/concepts/commitments)
- [Heartbeat](/gateway/heartbeat)
- [Scheduled tasks](/automation/cron-jobs)

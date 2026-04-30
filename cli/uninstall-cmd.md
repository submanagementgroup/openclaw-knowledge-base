---
domain: cli
topic: "openclaw uninstall: Remove Gateway Service and Local Data"
type: procedure
keywords:
  - openclaw uninstall
  - uninstall gateway
  - remove openclaw
  - dry-run uninstall
  - remove service
  - remove state
  - remove workspace
source: cli/uninstall.md
related:
  - cli/cli-overview
  - install/updating
---

`openclaw uninstall` removes the gateway service and/or local data. The CLI binary itself is not removed. Use `--dry-run` to preview actions before committing.

## Options

- `--service`: remove the gateway service
- `--state`: remove state and config
- `--workspace`: remove workspace directories
- `--app`: remove the macOS app
- `--all`: remove service, state, workspace, and app
- `--yes`: skip confirmation prompts
- `--non-interactive`: disable prompts; requires `--yes`
- `--dry-run`: print actions without removing files

## Examples

```bash
# Preview what would be removed
openclaw uninstall --dry-run

# Remove everything with confirmation
openclaw uninstall --all --yes

# Remove service only (non-interactive)
openclaw uninstall --service --yes --non-interactive

# Remove state and workspace (non-interactive)
openclaw uninstall --state --workspace --yes --non-interactive

# Full removal
openclaw uninstall --all --yes
```

## Best Practice: Backup First

Run `openclaw backup create` before removing state or workspaces to retain a restorable snapshot:

```bash
openclaw backup create
openclaw uninstall
```

## Notes

- `--all` is shorthand for removing service, state, workspace, and app together.
- `--non-interactive` requires `--yes`.
- The CLI binary remains after uninstall; reinstall via npm to get it back.

## Related

- [CLI overview](/cli)
- [Uninstall and updating](/install/updating)

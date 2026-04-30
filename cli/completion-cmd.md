---
domain: cli
topic: "openclaw completion: Generate and Install Shell Completion Scripts"
type: procedure
keywords:
  - openclaw completion
  - shell completion
  - zsh completion
  - bash completion
  - fish completion
  - powershell completion
  - tab completion
source: cli/completion.md
related:
  - cli/cli-overview
---

`openclaw completion` generates shell completion scripts for zsh, bash, fish, or PowerShell and optionally installs them into your shell profile.

## Usage

```bash
openclaw completion
openclaw completion --shell zsh
openclaw completion --install
openclaw completion --shell fish --install
openclaw completion --write-state
openclaw completion --shell bash --write-state
```

## Options

- `-s, --shell <shell>`: shell target (`zsh`, `bash`, `powershell`, `fish`; default: `zsh`)
- `-i, --install`: install completion by adding a source line to your shell profile
- `--write-state`: write completion script(s) to `$OPENCLAW_STATE_DIR/completions` without printing to stdout
- `-y, --yes`: skip install confirmation prompts

## How It Works

- `--install` writes a small "OpenClaw Completion" block into your shell profile and points it at the cached script.
- Without `--install` or `--write-state`, the command prints the script to stdout for manual installation.
- Completion generation eagerly loads command trees so nested subcommands are fully included.

## Quick Install (Recommended)

```bash
# Install for zsh (default)
openclaw completion --install

# Install for bash
openclaw completion --shell bash --install

# Install for fish
openclaw completion --shell fish --install
```

## Related

- [CLI overview](/cli)

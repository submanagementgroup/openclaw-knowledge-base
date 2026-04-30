---
domain: cli
topic: "openclaw clawbot: Legacy Alias Namespace for QR Commands"
type: reference
keywords:
  - clawbot
  - legacy alias
  - openclaw clawbot qr
  - backwards compatibility
  - migration
source: cli/clawbot.md
related:
  - cli/cli-overview
---

`openclaw clawbot` is a legacy alias namespace kept for backwards compatibility. The only current supported alias is `openclaw clawbot qr`, which is equivalent to `openclaw qr`.

## Supported Aliases

- `openclaw clawbot qr` → same behavior as `openclaw qr`

## Migration to Modern Commands

Prefer modern top-level commands directly:

```bash
# Old form (still works)
openclaw clawbot qr

# Preferred modern form
openclaw qr
```

The `clawbot` namespace exists only for backwards compatibility with older scripts. New scripts and documentation should use the direct top-level commands.

## Related

- [CLI overview](/cli)

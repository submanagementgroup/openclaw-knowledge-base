---
domain: cli
topic: "openclaw docs: Search the Live Docs Index from the Terminal"
type: reference
keywords:
  - openclaw docs
  - docs search
  - documentation search
  - live docs index
  - cli docs
source: cli/docs.md
related:
  - cli/cli-overview
---

`openclaw docs` searches the live OpenClaw documentation index from the terminal. Run it with no arguments to open the live docs search entrypoint.

## Usage

```bash
openclaw docs
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

## Arguments

- `[query...]`: search terms to send to the live docs index

## Notes

- With no query, `openclaw docs` opens the live docs search entrypoint.
- Multi-word queries are passed through as one search request.
- Useful for quickly looking up config keys, tool names, or concepts without leaving the terminal.

## Related

- [CLI overview](/cli)

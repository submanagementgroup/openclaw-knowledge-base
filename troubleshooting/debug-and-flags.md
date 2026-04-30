---
domain: troubleshooting
topic: "Node Debug and Diagnostic Flags: Debugging Node Issues and Gateway Diagnostics"
type: troubleshooting
keywords:
  - node debug
  - diagnostic flags
  - debug flags
  - node issue
  - gateway diagnostics
related:
  - gateway/health-diagnostics-logging
  - nodes/nodes-overview
source:
  - debug/node-issue.md
  - diagnostics/flags.md
---

Node debug guide and diagnostic flags.

## Node Issue Debugging

# Node + tsx "\_\_name is not a function" crash

## Summary

Running OpenClaw via Node with `tsx` fails at startup with:

```
[openclaw] Failed to start CLI: TypeError: __name is not a function
    at createSubsystemLogger (.../src/logging/subsystem.ts:203:25)
    at .../src/agents/auth-profiles/constants.ts:25:20
```

This began after switching dev scripts from Bun to `tsx` (commit `2871657e`, 2026-01-06). The same runtime path worked with Bun.

## Environment

- Node: v25.x (observed on v25.3.0)
- tsx: 4.21.0
- OS: macOS (repro also likely on other platforms that run Node 25)

## Repro (Node-only)

```bash
# in repo root
node --version
pnpm install
node --import tsx src/entry.ts status
```

## Minimal repro in repo

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

## Node version check

- Node 25.3.0: fails
- Node 22.22.0 (Homebrew `node@22`): fails
- Node 24: not installed here yet; needs verification

## Notes / hypothesis

- `tsx` uses esbuild to transform TS/ESM. esbuild’s `keepNames` emits a `__name` helper and wraps function definitions with `__name(...)`.
- The crash indicates `__name` exists but is not a function at runtime, which implies the helper is missing or overwritten for this module in the Node 25 loader path.
- Similar `__name` helper issues have been reported in other esbuild consumers when the helper is missing or rewritten.

## Regression history

- `2871657e` (2026-01-06): scripts changed from Bun to tsx to make Bun optional.
- Before that (Bun path), `openclaw status` and `gateway:watch` worked.

## Workarounds

- Use Bun for dev scripts (current temporary revert).
- Use `tsgo` for repo type checking, then run the built output:

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- Historical note: `tsc` was used here while debugging this Node/tsx issue, but repo type-check lanes now use `tsgo`.
- Disable esbuild keepNames in the TS loader if possible (prevents `__name` helper insertion); tsx does not currently expose this.
- Test Node LTS (22/24) with `tsx` to see if the issue is Node 25–specific.

## References

- [https://opennext.js.org/cloudflare/howtos/keep_names](https://opennext.js.org/cloudflare/howtos/keep_names)
- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## Next steps

- Repro on Node 22/24 to confirm Node 25 regression.
- Test `tsx` nightly or pin to earlier version if a known regression exists.
- If reproduces on Node LTS, file a minimal repro upstream with the `__name` stack trace.

## Related

- [Node.js install](/install/node)
- [Gateway troubleshooting](/gateway/troubleshooting)

## Diagnostic Flags

Diagnostics flags let you enable targeted debug logs without turning on verbose logging everywhere. Flags are opt-in and have no effect unless a subsystem checks them.

## How it works

- Flags are strings (case-insensitive).
- You can enable flags in config or via an env override.
- Wildcards are supported:
  - `telegram.*` matches `telegram.http`
  - `*` enables all flags

## Enable via config

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Multiple flags:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "gateway.*"]
  }
}
```

Restart the gateway after changing flags.

## Env override (one-off)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload
```

Disable all flags:

```bash
OPENCLAW_DIAGNOSTICS=0
```

## Timeline artifacts

The `timeline` flag writes structured startup and runtime timing events for
external QA harnesses:

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

You can also enable it in config:

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

The timeline file path still comes from
`OPENCLAW_DIAGNOSTICS_TIMELINE_PATH`. When `timeline` is enabled only from
config, the earliest config-loading spans are not emitted because OpenClaw has
not read config yet; subsequent startup spans use the config flag.

`OPENCLAW_DIAGNOSTICS=1`, `OPENCLAW_DIAGNOSTICS=all`, and
`OPENCLAW_DIAGNOSTICS=*` also enable the timeline because they enable every
diagnostics flag. Prefer `timeline` when you only want the JSONL timing
artifact.

Timeline records use the `openclaw.diagnostics.v1` envelope. Events can include
process ids, phase names, span names, durations, plugin ids, dependency counts,
event-loop delay samples, provider operation names, child-process exit state,
and startup error names/messages. Treat timeline files as local diagnostics
artifacts; review them before sharing outside your machine.

## Where logs go

Flags emit logs into the standard diagnostics log file. By default:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

If you set `logging.file`, use that path instead. Logs are JSONL (one JSON object per line). Redaction still applies based on `logging.redactSensitive`.

## Extract logs

Pick the latest log file:

```bash
ls -t /tmp/openclaw/openclaw-*.log | head -n 1
```

Filter for Telegram HTTP diagnostics:

```bash
rg "telegram http error" /tmp/openclaw/openclaw-*.log
```

Or tail while reproducing:

```bash
tail -f /tmp/openclaw/openclaw-$(date +%F).log | rg "telegram http error"
```

For remote gateways, you can also use `openclaw logs --follow` (see [/cli/logs](/cli/logs)).

## Notes

- If `logging.level` is set higher than `warn`, these logs may be suppressed. Default `info` is fine.
- Flags are safe to leave enabled; they only affect log volume for the specific subsystem.
- Use [/logging](/logging) to change log destinations, levels, and redaction.

## Related

- [Gateway diagnostics](/gateway/diagnostics)
- [Gateway troubleshooting](/gateway/troubleshooting)

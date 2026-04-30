---
domain: tools
topic: "Exec Approvals: Human Confirmation Gates for Shell Command Execution"
type: procedure
keywords:
  - exec approvals
  - command approvals
  - approval gates
  - exec-approvals
  - human confirmation
  - allow patterns
related:
  - tools/exec
  - gateway/sandboxing
  - concepts/agent-loop
source:
  - tools/exec-approvals.md
  - tools/exec-approvals-advanced.md
---

Exec approvals require human confirmation before running shell commands. Configure approval gates to control what the agent can execute without supervision.

Exec approvals are the **companion app / node host guardrail** for letting
a sandboxed agent run commands on a real host (`gateway` or `node`). A
safety interlock: commands are allowed only when policy + allowlist +
(optional) user approval all agree. Exec approvals stack **on top of**
tool policy and elevated gating (unless elevated is set to `full`, which
skips approvals).

Effective policy is the **stricter** of `tools.exec.*` and approvals
defaults; if an approvals field is omitted, the `tools.exec` value is
used. Host exec also uses local approvals state on that machine — a
host-local `ask: "always"` in `~/.openclaw/exec-approvals.json` keeps
prompting even if session or config defaults request `ask: "on-miss"`.

## Inspecting the effective policy

| Command                                                          | What it shows                                                                          |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | Requested policy, host policy sources, and the effective result.                       |
| `openclaw exec-policy show`                                      | Local-machine merged view.                                                             |
| `openclaw exec-policy set` / `preset`                            | Synchronize the local requested policy with the local host approvals file in one step. |

When a local scope requests `host=node`, `exec-policy show` reports that
scope as node-managed at runtime instead of pretending the local
approvals file is the source of truth.

If the companion app UI is **not available**, any request that would
normally prompt is resolved by the **ask fallback** (default: `deny`).

Native chat approval clients can seed channel-specific affordances on the
pending approval message. For example, Matrix seeds reaction shortcuts
(`✅` allow once, `❌` deny, `♾️` allow always) while still leaving
`/approve ...` commands in the message as a fallback.

## Where it applies

Exec approvals are enforced locally on the execution host:

- **Gateway host** → `openclaw` process on the gateway machine.
- **Node host** → node runner (macOS companion app or headless node host).

### Trust model

- Gateway-authenticated callers are trusted operators for that Gateway.
- Paired nodes extend that trusted operator capability onto the node host.
- Exec approvals reduce accidental execution risk, but are **not** a per-user auth boundary.
- Approved node-host runs bind canonical execution context: canonical cwd, exact argv, env binding when present, and pinned executable path when applicable.
- For shell scripts and direct interpreter/runtime file invocations, OpenClaw also tries to bind one concrete local file operand. If that bound file changes after approval but before execution, the run is denied instead of executing drifted content.
- File binding is intentionally best-effort, **not** a complete semantic model of every interpreter/runtime loader path. If approval mode cannot identify exactly one concrete local file to bind, it refuses to mint an approval-backed run instead of pretending full coverage.

### macOS split

- The **node host service** forwards `system.run` to the **macOS app** over local IPC.
- The **macOS app** enforces approvals and executes the command in UI context.

## Settings and storage

Approvals live in a local JSON file on the execution host:

```text
~/.openclaw/exec-approvals.json
```

Example schema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "source": "allow-always",
          "commandText": "rg -n TODO",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## Policy knobs

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` — block all host exec requests.
  - `allowlist` — allow only allowlisted commands.
  - `full` — allow everything (equivalent to elevated).

</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  - `off` — never prompt.
  - `on-miss` — prompt only when the allowlist does not match.
  - `always` — prompt on every command. `allow-always` durable trust does **not** suppress prompts when effective ask mode is `always`.

</ParamField>

### `

## Advanced Approval Configuration

Advanced exec-approval topics: the `safeBins` fast-path, interpreter/runtime
binding, and approval-forwarding to chat channels (including native delivery).
For the core policy and approval flow, see [Exec approvals](/tools/exec-approvals).

## Safe bins (stdin-only)

`tools.exec.safeBins` defines a small list of **stdin-only** binaries (for
example `cut`) that can run in allowlist mode **without** explicit allowlist
entries. Safe bins reject positional file args and path-like tokens, so they
can only operate on the incoming stream. Treat this as a narrow fast-path for
stream filters, not a general trust list.

Do **not** add interpreter or runtime binaries (for example `python3`, `node`,
`ruby`, `bash`, `sh`, `zsh`) to `safeBins`. If a command can evaluate code,
execute subcommands, or read files by design, prefer explicit allowlist entries
and keep approval prompts enabled. Custom safe bins must define an explicit
profile in `tools.exec.safeBinProfiles.<bin>`.

Default safe bins:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` and `sort` are not in the default list. If you opt in, keep explicit
allowlist entries for their non-stdin workflows. For `grep` in safe-bin mode,
provide the pattern with `-e`/`--regexp`; positional pattern form is rejected
so file operands cannot be smuggled as ambiguous positionals.

### Argv validation and denied flags

Validation is deterministic from argv shape only (no host filesystem existence
checks), which prevents file-existence oracle behavior from allow/deny
differences. File-oriented options are denied for default safe bins; long
options are validated fail-closed (unknown flags and ambiguous abbreviations are
rejected).

Denied flags by safe-bin profile:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Safe bins also force argv tokens to be treated as **literal text** at execution
time (no globbing and no `$VARS` expansion) for stdin-only segments, so patterns
like `*` or `$HOME/...` cannot be used to smuggle file reads.

### Trusted binary directories

Safe bins must resolve from trusted binary directories (system defaults plus
optional `tools.exec.safeBinTrustedDirs`). `PATH` entries are never auto-trusted.
Default trusted directories are intentionally minimal: `/bin`, `/usr/bin`. If
your safe-bin executable lives in package-manager/user paths (for example
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), add them
explicitly to `tools.exec.safeBinTrustedDirs`.

### Shell chaining, wrappers, and multiplexers

Shell chaining (`&&`, `||`, `;`) is allowed when every top-level segment
satisfies the allowlist (including safe bins or skill auto-allow). Redirections
remain unsupported in allowlist mode. Command substitution (`$()` / backticks) is
rejected during allowlist parsing, including inside double quotes; use single
quotes if you need literal `$()` text.

On macOS companion-app approvals, raw shell text containing shell control or
expansion syntax (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) is
treated as an allowlist miss unless the shell binary itself is allowlisted.

For shell wrappers (`bash|sh|zsh ... -c/-lc`), request-scoped env overrides are
reduced to a small explicit allowlist (`TERM`, `LANG`, `LC_*`, `COLORTERM`,
`NO_COLOR`, `FORCE_COLOR`).

For `allow-always` decisions in allowlist mode, known dispatch wrappers (`env`,
`nice`, `nohup`, `stdbuf`, `timeout`) persist the inner executable path instead
of the wrapper path. Shell multiplexers (`busybox`, `toybox`) are unwrapped for
shell applets (`sh`, `ash

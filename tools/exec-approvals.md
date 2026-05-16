---
domain: tools
topic: "Exec Approvals: Command Approval Policies"
type: reference
keywords:
  - exec approvals
  - command approvals
  - approval policy
  - auto-approve
  - exec policy
source: tools/exec-approvals.md
---

Exec approvals are the **companion app / node host guardrail** for letting
a sandboxed agent run commands on a real host (`gateway` or `node`). A
safety interlock: commands are allowed only when policy + allowlist +
(optional) user approval all agree. Exec approvals stack **on top of**
tool policy and elevated gating (unless elevated is set to `full`, which
skips approvals).

> **Note:** Effective policy is the **stricter** of `tools.exec.*` and approvals
defaults; if an approvals field is omitted, the `tools.exec` value is
used. Host exec also uses local approvals state on that machine - a
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

> **Note:** Native chat approval clients can seed channel-specific affordances on the
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
- Exec approvals reduce accidental execution risk, but are **not** a per-user auth boundary or filesystem read-only policy.
- Once approved, a command can mutate files according to the selected host or sandbox filesystem permissions.
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


  - `deny` - block all host exec requests.
  - `allowlist` - allow only allowlisted commands.
  - `full` - allow everything (equivalent to elevated).



### `exec.ask`


  - `off` - never prompt.
  - `on-miss` - prompt only when the allowlist does not match.
  - `always` - prompt on every command. `allow-always` durable trust does **not** suppress prompts when effective ask mode is `always`.



### `askFallback`


  Resolution when a prompt is required but no UI is reachable.

- `deny` - block.
- `allowlist` - allow only if allowlist matches.
- `full` - allow.



### `tools.exec.strictInlineEval`


  When `true`, OpenClaw treats inline code-eval forms as approval-only
  even if the interpreter binary itself is allowlisted. Defense-in-depth
  for interpreter loaders that do not map cleanly to one stable file
  operand.


Examples that strict mode catches:

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

In strict mode these commands still need explicit approval, and
`allow-always` does not persist new allowlist entries for them
automatically.

### `tools.exec.commandHighlighting`


  Controls only presentation in exec approval prompts. When enabled,
  OpenClaw may attach parser-derived command spans so Web approval
  prompts can highlight command tokens. Set it to `true` to enable
  command text highlighting.


This setting does **not** change `security`, `ask`, allowlist matching,
strict inline-eval behavior, approval forwarding, or command execution.
It can be set globally under `tools.exec.commandHighlighting` or per
agent under `agents.list[].tools.exec.commandHighlighting`.

## YOLO mode (no-approval)

If you want host exec to run without approval prompts, you must open
**both** policy layers - requested exec policy in OpenClaw config
(`tools.exec.*`) **and** host-local approvals policy in
`~/.openclaw/exec-approvals.json`.

YOLO is the default host behavior unless you tighten it explicitly:

| Layer                 | YOLO setting               |
| --------------------- | -------------------------- |
| `tools.exec.security` | `full` on `gateway`/`node` |
| `tools.exec.ask`      | `off`                      |
| Host `askFallback`    | `full`                     |

> **Note:** **Important distinctions:**

- `tools.exec.host=auto` chooses **where** exec runs: sandbox when available, otherwise gateway.
- YOLO chooses **how** host exec is approved: `security=full` plus `ask=off`.
- In YOLO mode, OpenClaw does **not** add a separate heuristic command-obfuscation approval gate or script-preflight rejection layer on top of the configured host exec policy.
- `auto` does not make gateway routing a free override from a sandboxed session. A per-call `host=node` request is allowed from `auto`; `host=gateway` is only allowed from `auto` when no sandbox runtime is active. For a stable non-auto default, set `tools.exec.host` or use `/exec host=...` explicitly.



CLI-backed providers that expose their own noninteractive permission mode
can follow this policy. Claude CLI adds
`--permission-mode bypassPermissions` when OpenClaw's requested exec
policy is YOLO. Override that backend behavior with explicit Claude args
under `agents.defaults.cliBackends.claude-cli.args` / `resumeArgs` -
for example `--permission-mode default`, `acceptEdits`, or
`bypassPermissions`.

If you want a more conservative setup, tighten either layer back to
`allowlist` / `on-miss` or `deny`.

### Persistent gateway-host "never prompt" setup

**Set the requested config policy**

```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.security full
    openclaw config set tools.exec.ask off
    openclaw gateway restart
    ```
  
**Match the host approvals file**

```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  
### Local shortcut

```bash
openclaw exec-policy preset yolo
```

That local shortcut updates both:

- Local `tools.exec.host/security/ask`.
- Local `~/.openclaw/exec-approvals.json` defaults.

It is intentionally local-only. To change gateway-host or node-host
approvals remotely, use `openclaw approvals set --gateway` or
`openclaw approvals set --node <id|name|ip>`.

### Node host

For a node host, apply the same approvals file on that node instead:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

> **Note:** **Local-only limitations:**

- `openclaw exec-policy` does not synchronize node approvals.
- `openclaw exec-policy set --host node` is rejected.
- Node exec approvals are fetched from the node at runtime, so node-targeted updates must use `openclaw approvals --node ...`.



### Session-only shortcut

- `/exec security=full ask=off` changes only the current session.
- `/elevated full` is a break-glass shortcut that also skips exec approvals for that session.

If the host approvals file stays stricter than config, the stricter host
policy still wins.

## Allowlist (per agent)

Allowlists are **per agent**. If multiple agents exist, switch which agent
you are editing in the macOS app. Patterns are glob matches.

Patterns can be resolved binary path globs or bare command-name globs.
Bare names match only commands invoked through `PATH`, so `rg` can match
`/opt/homebrew/bin/rg` when the command is `rg`, but **not** `./rg` or
`/tmp/rg`. Use a path glob when you want to trust one specific binary
location.

Legacy `agents.default` entries are migrated to `agents.main` on load.
Shell chains such as `echo ok && pwd` still need every top-level segment
to satisfy allowlist rules.

Examples:

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### Restricting arguments with argPattern

Add `argPattern` when an allowlist entry should match a binary and a
specific argument shape. OpenClaw evaluates the regular expression
against the parsed command arguments, excluding the executable token
(`argv[0]`). For hand-authored entries, arguments are joined with a
single space, so anchor the pattern when you need an exact match.

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

That entry allows `python3 safe.py`; `python3 other.py` is an allowlist
miss. If a path-only entry for the same binary is also present, unmatched
arguments can still fall back to that path-only entry. Omit the path-only
entry when the goal is to restrict the binary to the declared arguments.

Entries saved by approval flows can use an internal separator format for
exact argv matching. Prefer the UI or approval flow to regenerate those
entries instead of hand-editing the encoded value. If OpenClaw cannot
parse argv for a command segment, entries with `argPattern` do not match.

Each allowlist entry supports:

| Field              | Meaning                                                       |
| ------------------ | ------------------------------------------------------------- |
| `pattern`          | Resolved binary path glob or bare command-name glob           |
| `argPat
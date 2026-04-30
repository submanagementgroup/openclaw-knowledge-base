---
domain: plugins
topic: "Skill Workshop Plugin: Creating, Testing, and Publishing Agent Skills"
type: procedure
keywords:
  - Skill Workshop
  - skill creation
  - skill plugin
  - skill testing
  - skill publishing
related:
  - tools/skills
  - plugins/building-plugins
source: plugins/skill-workshop.md
---

Skill Workshop is a plugin for creating, testing, and publishing skills. Skills are contextual instruction files for agents.

Skill Workshop is **experimental**. It is disabled by default, its capture
heuristics and reviewer prompts may change between releases, and automatic
writes should be used only in trusted workspaces after reviewing pending-mode
output first.

Skill Workshop is procedural memory for workspace skills. It lets an agent turn
reusable workflows, user corrections, hard-won fixes, and recurring pitfalls
into `SKILL.md` files under:

```text
<workspace>/skills/<skill-name>/SKILL.md
```

This is different from long-term memory:

- **Memory** stores facts, preferences, entities, and past context.
- **Skills** store reusable procedures the agent should follow on future tasks.
- **Skill Workshop** is the bridge from a useful turn to a durable workspace
  skill, with safety checks and optional approval.

Skill Workshop is useful when the agent learns a procedure such as:

- how to validate externally sourced animated GIF assets
- how to replace screenshot assets and verify dimensions
- how to run a repo-specific QA scenario
- how to debug a recurring provider failure
- how to repair a stale local workflow note

It is not intended for:

- facts like “the user likes blue”
- broad autobiographical memory
- raw transcript archiving
- secrets, credentials, or hidden prompt text
- one-off instructions that will not repeat

## Default state

The bundled plugin is **experimental** and **disabled by default** unless it is
explicitly enabled in `plugins.entries.skill-workshop`.

The plugin manifest does not set `enabledByDefault: true`. The `enabled: true`
default inside the plugin config schema applies only after the plugin entry has
already been selected and loaded.

Experimental means:

- the plugin is supported enough for opt-in testing and dogfooding
- proposal storage, reviewer thresholds, and capture heuristics can evolve
- pending approval is the recommended starting mode
- auto apply is for trusted personal/workspace setups, not shared or hostile
  input-heavy environments

## Enable

Minimal safe config:

```json5
{
  plugins: {
    entries: {
      "skill-workshop": {
        enabled: true,
        config: {
          autoCapture: true,
          approvalPolicy: "pending",
          reviewMode: "hybrid",
        },
      },
    },
  },
}
```

With this config:

- the `skill_workshop` tool is available
- explicit reusable corrections are queued as pending proposals
- threshold-based reviewer passes can propose skill updates
- no skill file is written until a pending proposal is applied

Use automatic writes only in trusted workspaces:

```json5
{
  plugins: {
    entries: {
      "skill-workshop": {
        enabled: true,
        config: {
          autoCapture: true,
          approvalPolicy: "auto",
          reviewMode: "hybrid",
        },
      },
    },
  },
}
```

`approvalPolicy: "auto"` still uses the same scanner and quarantine path. It
does not apply proposals with critical findings.

## Configuration

| Key                  | Default     | Range / values                              | Meaning                                                              |
| -------------------- | ----------- | ------------------------------------------- | -------------------------------------------------------------------- |
| `enabled`            | `true`      | boolean                                     | Enables the plugin after the plugin entry is loaded.                 |
| `autoCapture`        | `true`      | boolean                                     | Enables post-turn capture/review on successful agent turns.          |
| `approvalPolicy`     | `"pending"` | `"pending"`, `"auto"`                       | Queue proposals or write safe proposals automatically.               |
| `reviewMode`         | `"hybrid"`  | `"off"`, `"heuristic"`, `"llm"`, `"hybrid"` | Chooses explicit correction capture, LLM reviewer, both, or neither. |
| `reviewInterval`     | `15`        | `1..200`                                    | Run reviewer after this many successful turns.                       |
| `reviewMinToolCalls` | `8`         | `1..500`                                    | Run reviewer after this many observed tool calls.                    |
| `reviewTimeoutMs`    | `45000`     | `5000..180000`                              | Timeout for the embedded reviewer run.                               |
| `maxPending`         | `50`        | `1..200`                                    | Max pending/quarantined proposals kept per workspace.                |
| `maxSkillBytes`      | `40000`     | `1024..200000`                              | Max generated skill/support file size.                               |

Recommended profiles:

```json5
// Conservative: explicit tool use only, no automatic capture.
{
  autoCapture: false,
  approvalPolicy: "pending",
  reviewMode: "off",
}
```

```json5
// Review-first: capture automatically, but require approval.
{
  autoCapture: true,
  approvalPolicy: "pending",
  reviewMode: "hybrid",
}
```

```json5
// Trusted automation: write safe proposals immediately.
{
  autoCapture: true,
  approvalPolicy: "auto",
  reviewMode: "hybrid",
}
```

```json5
// Low-cost: no reviewer LLM call, only explicit correction phrases.
{
  autoCapture: true,
  approvalPolicy: "pending",
  reviewMode: "heuristic",
}
```

## Capture paths

Skill Workshop has three capture paths.

### Tool suggestions

The model can call `skill_workshop` directly when it sees a reusable procedure
or when the user asks it to save/update a skill.

This is the most explicit path and works even with `autoCapture: false`.

### Heuristic capture

When `autoCapture` is enabled and `reviewMode` is `heuristic` or `hybrid`, the
plugin scans successful turns for explicit user correction phrases:

- `next time`
- `from now on`
- `remember to`
- `make sure to`
- `always ... use/check/verify/record/save/prefer`
- `prefer ... when/for/instead/use`
- `when asked`

The heuristic creates a proposal from the latest matching user instruction. It
uses topic hints to choose skill names for common workflows:

- animated GIF tasks -> `animated-gif-workflow`
- screenshot or asset tasks -> `screenshot-asset-workflow`
- QA or scenario tasks -> `qa-scenario-workflow`
- GitHub PR tasks -> `github-pr-workflow`
- fallback -> `learned-workflows`

Heuristic capture is intentionally narrow. It is for clear corrections and
repeatable process notes, not for general transcript summarization.

### LLM reviewer

When `autoCapture` is enabled and `reviewMode` is `llm` or `hybrid`, the plugin
runs a compact embedded reviewer after thresholds are reached.

The reviewer receives:

- the recent transcript text, capped to the last 12,000 characters
- up to 12 existing workspace skills
- up to 2,000 characters from each existing skill
- JSON-only instructions

The reviewer has no tools:

- `disableTools: true`
- `toolsAllow: []`
- `disableMessageTool: true`

The reviewer returns either `{ "action": "none" }` or one proposal. The `action` field is `create`, `append`, or `replace` — prefer `append`/`replace` when a relevant skill already exists; use `create` only when no existing skill fits.

Example `create`:

```json
{
  "action": "create",
  "skillName": "media-asset-qa",
  "title": "Media Asset QA",
  "reason": "Reusable animated media acceptance workflow",
  "description": "Validate externally sourced animated media before product use.",
  "body": "## Workflow\n\n- Verify true animation.\n- Record attribution.\n- Store a local approved copy.\n- Verify in product UI before final reply."
}
```

`append` adds `section` + `body`. `replace` swaps `oldText` for `newText` in the named skill.

## Proposal lifecycle

Every generated update becomes a proposal with:

- `id`
- `createdAt`
- `updatedAt`
- `workspaceDir`
- optional `agentId`
- optional `sessionId`
- `skillName`
- `title`
- `reason`
- `source`: `tool`, `agent_end`, or `reviewer`
- `status`
- `change`
- optional `scanFindings`
- optional `quarantineReason`

Proposal statuses:

- `pending` - waiting for approval
- `applied` - written to `<workspace>/skills`
- `rejected` - rejected by operator/model
- `quarantined` - blocked by critical scanner findings

State is stored per workspace under the Gateway state directory:

```text
<stateDir>/skill-workshop/<workspace-hash>.json
```

Pending and quarantined proposals are deduplicated by skill name and change
payload. The store keeps the newest pending/quarantined proposals up to
`maxPending`.

## Tool reference

The plugin registers one agent tool:

```text
skill_workshop
```

### `status`

Count proposals by state for the active workspace.

```json
{ "action": "status" }
```

Result shape:

```json
{
  "workspaceDir": "/path/to/workspace",
  "pending": 1,
  "quarantined": 0,
  "applied": 3,
  "rejected": 0
}
```

### `list_pending`

List pending proposals.

```json
{ "action": "list_pending" }
```

To list another status:

```json
{ "action": "list_pending", "status": "applied" }
```

Valid `status` valu

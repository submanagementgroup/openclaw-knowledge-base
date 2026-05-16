---
domain: automation
topic: "Cron Jobs: Scheduled Automation"
type: procedure
keywords:
  - cron jobs
  - cron
  - scheduled tasks
  - automation
  - cron schedule
  - periodic tasks
source: 
  - automation/cron-jobs.md
  - automation/cron-vs-heartbeat.md
related:
  - gateway/heartbeat
---

Cron is the Gateway's built-in scheduler. It persists jobs, wakes the agent at the right time, and can deliver output back to a chat channel or webhook endpoint.

## Quick start

**Add a one-shot reminder**

```bash
    openclaw cron add \
      --name "Reminder" \
      --at "2026-02-01T16:00:00Z" \
      --session main \
      --system-event "Reminder: check the cron docs draft" \
      --wake now \
      --delete-after-run
    ```
  
**Check your jobs**

```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  
**See run history**

```bash
    openclaw cron runs --id <job-id>
    ```
  
## How cron works

- Cron runs **inside the Gateway** process (not inside the model).
- Job definitions persist at `~/.openclaw/cron/jobs.json` so restarts do not lose schedules.
- Runtime execution state persists next to it in `~/.openclaw/cron/jobs-state.json`. If you track cron definitions in git, track `jobs.json` and gitignore `jobs-state.json`.
- After the split, older OpenClaw versions can read `jobs.json` but may treat jobs as fresh because runtime fields now live in `jobs-state.json`.
- When `jobs.json` is edited while the Gateway is running or stopped, OpenClaw compares the changed schedule fields with pending runtime slot metadata and clears stale `nextRunAtMs` values. Pure formatting or key-order-only rewrites preserve the pending slot.
- All cron executions create [background task](/automation/tasks) records.
- On Gateway startup, overdue isolated agent-turn jobs are rescheduled out of the channel-connect window instead of replaying immediately, so Discord/Telegram startup and native-command setup stay responsive after restarts.
- One-shot jobs (`--at`) auto-delete after success by default.
- Isolated cron runs best-effort close tracked browser tabs/processes for their `cron:<jobId>` session when the run completes, so detached browser automation does not leave orphaned processes behind.
- Isolated cron runs that receive the narrow cron self-cleanup grant can still read scheduler status, a self-filtered list of their current job, and that job's run history, so status/heartbeat checks can inspect their own schedule without gaining broader cron mutation access.
- Isolated cron runs also guard against stale acknowledgement replies. If the first result is just an interim status update (`on it`, `pulling everything together`, and similar hints) and no descendant subagent run is still responsible for the final answer, OpenClaw re-prompts once for the actual result before delivery.
- Isolated cron runs prefer structured execution-denial metadata from the embedded run, then fall back to known final summary/output markers such as `SYSTEM_RUN_DENIED` and `INVALID_REQUEST`, so a blocked command is not reported as a green run.
- Isolated cron runs also treat run-level agent failures as job errors even when no reply payload is produced, so model/provider failures increment error counters and trigger failure notifications instead of clearing the job as successful.
- When an isolated agent-turn job reaches `timeoutSeconds`, cron aborts the underlying agent run and gives it a short cleanup window. If the run does not drain, Gateway-owned cleanup force-clears that run's session ownership before cron records the timeout, so queued chat work is not left behind a stale processing session.
- If an isolated agent-turn stalls before the runner starts or before the first model call, cron records a phase-specific timeout such as `setup timed out before runner start` or `stalled before first model call (last phase: context-engine)`. These watchdogs cover embedded providers and CLI-backed providers before their external CLI process is actually started, and are capped independently from long `timeoutSeconds` values so cold-start/auth/context failures surface quickly instead of waiting for the full job budget.

<a id="maintenance"></a>

> **Note:** Task reconciliation for cron is runtime-owned first, durable-history-backed second: an active cron task stays live while the cron runtime still tracks that job as running, even if an old child session row still exists. Once the runtime stops owning the job and the 5-minute grace window expires, maintenance checks persisted run logs and job state for the matching `cron:<jobId>:<startedAt>` run. If that durable history shows a terminal result, the task ledger is finalized from it; otherwise Gateway-owned maintenance can mark the task `lost`. Offline CLI audit can recover from durable history, but it does not treat its own empty in-process active-job set as proof that a Gateway-owned cron run is gone.


## Schedule types

| Kind    | CLI flag  | Description                                             |
| ------- | --------- | ------------------------------------------------------- |
| `at`    | `--at`    | One-shot timestamp (ISO 8601 or relative like `20m`)    |
| `every` | `--every` | Fixed interval                                          |
| `cron`  | `--cron`  | 5-field or 6-field cron expression with optional `--tz` |

Timestamps without a timezone are treated as UTC. Add `--tz America/New_York` for local wall-clock scheduling.

Recurring top-of-hour expressions are automatically staggered by up to 5 minutes to reduce load spikes. Use `--exact` to force precise timing or `--stagger 30s` for an explicit window.

### Day-of-month and day-of-week use OR logic

Cron expressions are parsed by [croner](https://github.com/Hexagon/croner). When both the day-of-month and day-of-week fields are non-wildcard, croner matches when **either** field matches — not both. This is standard Vixie cron behavior.

```
# Intended: "9 AM on the 15th, only if it's a Monday"
# Actual:   "9 AM on every 15th, AND 9 AM on every Monday"
0 9 15 * 1
```

This fires ~5–6 times per month instead of 0–1 times per month. OpenClaw uses Croner's default OR behavior here. To require both conditions, use Croner's `+` day-of-week modifier (`0 9 15 * +1`) or schedule on one field and guard the other in your job's prompt or command.

## Execution styles

| Style           | `--session` value   | Runs in                  | Best for                        |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| Main session    | `main`              | Next heartbeat turn      | Reminders, system events        |
| Isolated        | `isolated`          | Dedicated `cron:<jobId>` | Reports, background chores      |
| Current session | `current`           | Bound at creation time   | Context-aware recurring work    |
| Custom session  | `session:custom-id` | Persistent named session | Workflows that build on history |

### Main session vs isolated vs custom

**Main session** jobs enqueue a system event and optionally wake the heartbeat (`--wake now` or `--wake next-heartbeat`). Those system events do not extend daily/idle reset freshness for the target session. **Isolated** jobs run a dedicated agent turn with a fresh session. **Custom sessions** (`session:xxx`) persist context across runs, enabling workflows like daily standups that build on previous summaries.
  ### What 'fresh session' means for isolated jobs

For isolated jobs, "fresh session" means a new transcript/session id for each run. OpenClaw may carry safe preferences such as thinking/fast/verbose settings, labels, and explicit user-selected model/auth overrides, but it does not inherit ambient conversation context from an older cron row: channel/group routing, send or queue policy, elevation, origin, or ACP runtime binding. Use `current` or `session:<id>` when a recurring job should deliberately build on the same conversation context.
  ### Runtime cleanup

For isolated jobs, runtime teardown now includes best-effort browser cleanup for that cron session. Cleanup failures are ignored so the actual cron result still wins.

    Isolated cron runs also dispose any bundled MCP runtime instances created for the job through the shared runtime-cleanup path. This matches how main-session and custom-session MCP clients are torn down, so isolated cron jobs do not leak stdio child processes or long-lived MCP connections across runs.

  ### Subagent and Discord delivery

When isolated cron runs orchestrate subagents, delivery also prefers the final descendant output over stale parent interim text. If descendants are still running, OpenClaw suppresses that partial parent update instead of announcing it.

    For text-only Discord announce targets, OpenClaw sends the canonical final assistant text once instead of replaying both streamed/intermediate text payloads and the final answer. Media and structured Discord payloads are still delivered as separate payloads so attachments and components are not dropped.

  ### Payload options for isolated jobs


  Prompt text (required for isolated).


  Model override; uses the selected allowed model for the job.


  Thinking level override.


  Skip workspace bootstrap file injection.


  Restrict which tools the job can use, for example `--tools exec,read`.


`--model` uses the selected allowed model as that job's primary model. It is not the same as a chat-session `/model` override: configured fallback chains still apply when the job primary fails. If the requested model is not allowed or cannot be resolved, cron fails the run with an explicit validation error instead of silently falling back to the job's agent/default model selection.

Cron jobs can also carry payload-level `fallbacks`. When present, that list replaces the configured fallback chain for the job. Use `fallbacks: []` in the job payload/API when you want a strict cron run that tries only the selected model. If a job has `--model` but neither payload nor configured fallbacks, OpenClaw passes an explicit empty fallback override so the agent primary is not appended as a hidden extra retry target.

Model-selection precedence for isolated jobs is:

1. Gmail hook model override (when the run came f

## Cron vs Heartbeat

The decision guide for cron vs heartbeat lives under [Automation](/automation).

## Related

- [Scheduled tasks](/automation/cron-jobs)
- [Background tasks](/automation/tasks)

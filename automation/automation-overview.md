---
domain: automation
topic: "Automation Overview: Cron, Heartbeat, Hooks, Webhooks, Tasks, and Standing Orders"
type: concept
keywords:
  - automation
  - cron
  - heartbeat
  - hooks
  - webhooks
  - standing orders
  - background tasks
  - scheduled tasks
related:
  - automation/cron-jobs
  - automation/hooks
  - automation/tasks
  - automation/standing-orders
source:
  - automation/index.md
  - automation/taskflow.md
---

OpenClaw automation: scheduling (cron, heartbeat), event-driven hooks, background tasks, webhooks, and standing orders.

## Automation Methods Overview

| Method | Use Case |
|--------|----------|
| Cron jobs | Scheduled periodic work with full records |
| Heartbeat | Ambient agent check-in (lighter than cron) |
| Hooks | Event-driven: /new, /reset, gateway events |
| Webhooks | External HTTP triggers |
| Standing orders | Permanent autonomous programs |
| Background tasks | Track ACP, subagent, and cron runs |

OpenClaw runs work in the background through tasks, scheduled jobs, inferred
commitments, event hooks, and standing instructions. This page helps you choose
the right mechanism and understand how they fit together.

## Quick decision guide

| Use case                                | Recommended            | Why                                              |
| --------------------------------------- | ---------------------- | ------------------------------------------------ |
| Send daily report at 9 AM sharp         | Scheduled Tasks (Cron) | Exact timing, isolated execution                 |
| Remind me in 20 minutes                 | Scheduled Tasks (Cron) | One-shot with precise timing (`--at`)            |
| Run weekly deep analysis                | Scheduled Tasks (Cron) | Standalone task, can use different model         |
| Check inbox every 30 min                | Heartbeat              | Batches with other checks, context-aware         |
| Monitor calendar for upcoming events    | Heartbeat              | Natural fit for periodic awareness               |
| Check in after a mentioned interview    | Inferred Commitments   | Memory-like follow-up, no exact reminder request |
| Gentle care check-in after user context | Inferred Commitments   | Scoped to the same agent and channel             |
| Inspect status of a subagent or ACP run | Background Tasks       | Tasks ledger tracks all detached work            |
| Audit what ran and when                 | Background Tasks       | `openclaw tasks list` and `openclaw tasks audit` |
| Multi-step research then summarize      | Task Flow              | Durable orchestration with revision tracking     |
| Run a script on session reset           | Hooks                  | Event-driven, fires on lifecycle events          |
| Execute code on every tool call         | Plugin hooks           | In-process hooks can intercept tool calls        |
| Always check compliance before replying | Standing Orders        | Injected into every session automatically        |

### Scheduled Tasks (Cron) vs Heartbeat

| Dimension       | Scheduled Tasks (Cron)              | Heartbeat                             |
| --------------- | ----------------------------------- | ------------------------------------- |
| Timing          | Exact (cron expressions, one-shot)  | Approximate (default every 30 min)    |
| Session context | Fresh (isolated) or shared          | Full main-session context             |
| Task records    | Always created                      | Never created                         |
| Delivery        | Channel, webhook, or silent         | Inline in main session                |
| Best for        | Reports, reminders, background jobs | Inbox checks, calendar, notifications |

Use Scheduled Tasks (Cron) when you need precise timing or isolated execution. Use Heartbeat when the work benefits from full session context and approximate timing is fine.

## Core concepts

### Scheduled tasks (cron)

Cron is t

## Task Flow

Task Flow is the flow orchestration substrate that sits above [background tasks](/automation/tasks). It manages durable multi-step flows with their own state, revision tracking, and sync semantics while individual tasks remain the unit of detached work.

## When to use Task Flow

Use Task Flow when work spans multiple sequential or branching steps and you need durable progress tracking across gateway restarts. For single background operations, a plain [task](/automation/tasks) is sufficient.

| Scenario                              | Use                  |
| ------------------------------------- | -------------------- |
| Single background job                 | Plain task           |
| Multi-step pipeline (A then B then C) | Task Flow (managed)  |
| Observe externally created tasks      | Task Flow (mirrored) |
| One-shot reminder                     | Cron job             |

## Reliable scheduled workflow pattern

For recurring workflows such as market intelligence briefings, treat the schedule, orchestration, and reliability checks as separate layers:

1. Use [Scheduled Tasks](/automation/cron-jobs) for timing.
2. Use a persistent cron session when the workflow should build on prior context.
3. Use [Lobster](/tools/lobster) for deterministic steps, approval gates, and resume tokens.
4. Use Task Flow to track the multi-step run across child tasks, waits, retries, and gateway restarts.

Example cron shape:

```bash
openclaw cron add \
  --name "Market intelligence brief" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "Run the market-intel Lobster workflow. Verify source freshness before summarizing." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

Use `session:<id>` instead of `isolated` when the recurring workflow needs deliberate history, previous run summaries, or standing context. Use `isolated` when each run should start fresh and all required state is explicit in the workflow.

Inside the workflow, put reliability checks before the LLM summary step:

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

Recommended preflight checks:

- Browser availability and profile choice, for example `openclaw` for managed state or `user` when a signed-in Chrome session is required. See [Browser](/tools/browser).
- API credentials and quota for each source.
- Network reachability for required endpoints.
- Required tools enabled for the agent, such as `lobster`, `browser`, and `llm-task`.
- Failure destination configured for cron so prefl

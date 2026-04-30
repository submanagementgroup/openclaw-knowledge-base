---
domain: troubleshooting
topic: "GPT-5 and Codex Agentic Parity: Compatibility Notes and Maintainer Guidance"
type: reference
keywords:
  - GPT-5
  - Codex parity
  - agentic parity
  - GPT-5 compatibility
  - Codex compatibility
related:
  - providers/openai
  - plugins/codex-harness
source:
  - help/gpt55-codex-agentic-parity.md
  - help/gpt55-codex-agentic-parity-maintainers.md
---

GPT-5 and Codex agentic parity notes for OpenClaw maintainers.

# GPT-5.5 / Codex Agentic Parity in OpenClaw

OpenClaw already worked well with tool-using frontier models, but GPT-5.5 and Codex-style models were still underperforming in a few practical ways:

- they could stop after planning instead of doing the work
- they could use strict OpenAI/Codex tool schemas incorrectly
- they could ask for `/elevated full` even when full access was impossible
- they could lose long-running task state during replay or compaction
- parity claims against Claude Opus 4.6 were based on anecdotes instead of repeatable scenarios

This parity program fixes those gaps in four reviewable slices.

## What changed

### PR A: strict-agentic execution

This slice adds an opt-in `strict-agentic` execution contract for embedded Pi GPT-5 runs.

When enabled, OpenClaw stops accepting plan-only turns as “good enough” completion. If the model only says what it intends to do and does not actually use tools or make progress, OpenClaw retries with an act-now steer and then fails closed with an explicit blocked state instead of silently ending the task.

This improves the GPT-5.5 experience most on:

- short “ok do it” follow-ups
- code tasks where the first step is obvious
- flows where `update_plan` should be progress tracking rather than filler text

### PR B: runtime truthfulness

This slice makes OpenClaw tell the truth about two things:

- why the provider/runtime call failed
- whether `/elevated full` is actually available

That means GPT-5.5 gets better runtime signals for missing scope, auth refresh failures, HTML 403 auth failures, proxy issues, DNS or timeout failures, and blocked full-access modes. The model is less likely to hallucinate the wrong remediation or keep asking for a permission mode the runtime cannot provide.

### PR C: execution correctness

This slice improves two kinds of correctness:

- provider-owned OpenAI/Codex tool-schema compatibility
- replay and long-task liveness surfacing

The tool-compat work reduces schema friction for strict OpenAI/Codex tool registration, especially around parameter-free tools and strict object-root expectations. The replay/liveness work makes long-running tasks more observable, so paused, blocked, and abandoned states are visible instead of disappearing into generic failure text.

### PR D: parity harness

This slice adds the first-wave QA-lab parity pack so GPT-5.5 and Opus 4.6 can be exercised through the same scenarios and compared using shared evidence.

The parity pack is the proof layer. It does not change runtime behavior by itself.

After you have two `qa-suite-summary.json` artifacts, generate the release-gate comparison with:

```bash
pnpm openclaw qa parity-report \
  --repo-root . \
  --candidate-summary .artifacts/qa-e2e/gpt55/qa-suite-summary.json \
  --baseline-summary .artifacts/qa-e2e/opus46/qa-suite-summary.json \
  --output-dir .artifacts/qa-e2e/parity
```

That command writes:

- a human-readable Markdown report
- a machine-readable JSON verdict
- an explicit `pass` / `fail` gate result

## Why this improves GPT-5.5 in practice

Before this work, GPT-5.5 on OpenClaw could feel less agentic than Opus in real coding sessions because the runtime tolerated behaviors that are especially harmful for GPT-5-style models:

- commentary-only turns
- schema friction around tools
- vague permission feedback
- silent replay or compaction breakage

The goal is not to make GPT-5.5 imitate Opus. The goal is to give GPT-5.5 a runtime contract that rewards real progress, supplies cleaner tool and permission semantics, and turns failure modes into explicit machine- and human-readable states.

That changes the user experience from:

- “the model had a good plan but stopped”

to:

- “the model either acted, or OpenClaw surfaced the exact reason it could not”

## Before vs after for GPT-5.5 users

| Before this program                                                                            | After PR A-D                                                         

## Maintainers Notes

This note explains how to review the GPT-5.5 / Codex parity program as four merge units without losing the original six-contract architecture.

## Merge units

### PR A: strict-agentic execution

Owns:

- `executionContract`
- GPT-5-first same-turn follow-through
- `update_plan` as non-terminal progress tracking
- explicit blocked states instead of plan-only silent stops

Does not own:

- auth/runtime failure classification
- permission truthfulness
- replay/continuation redesign
- parity benchmarking

### PR B: runtime truthfulness

Owns:

- Codex OAuth scope correctness
- typed provider/runtime failure classification
- truthful `/elevated full` availability and blocked reasons

Does not own:

- tool schema normalization
- replay/liveness state
- benchmark gating

### PR C: execution correctness

Owns:

- provider-owned OpenAI/Codex tool compatibility
- parameter-free strict schema handling
- replay-invalid surfacing
- paused, blocked, and abandoned long-task state visibility

Does not own:

- self-elected continuation
- generic Codex dialect behavior outside provider hooks
- benchmark gating

### PR D: parity harness

Owns:

- first-wave GPT-5.5 vs Opus 4.6 scenario pack
- parity documentation
- parity report and release-gate mechanics

Does not own:

- runtime behavior changes outside QA-lab
- auth/proxy/DNS simulation inside the harness

## Mapping back to the original six contracts

| Original contract                        | Merge unit |
| ---------------------------------------- | ---------- |
| Provider transport/auth correctness      | PR B       |
| Tool contract/schema compatibility       | PR C       |
| Same-turn execution                      | PR A       |
| Permission truthfulness                  | PR B       |
| Replay/continuation/liveness correctness | PR C       |
| Benchmark/release gate                   | PR D       |

## Review order

1. PR A
2. PR B
3. PR C
4. PR D

PR D is the proof layer. It should not be the reason runtime-correctness PRs are delayed.

## What to look for

### PR A

- GPT-5 runs act or fail closed instead of stopping at commentary
- `update_plan` no longer looks like progress by itself
- behavior stays GPT-5-first and embedded-Pi scoped

### PR B

- auth/proxy/runtime failures stop collapsing into generic “model failed” handling
- `/elevated full` is only described as available when it is actually available
- blocked reasons are visible to both the model and the user-facing runtime

### PR C

- strict OpenAI/Codex tool registration behaves predictably
- parameter-free tools do not fail strict schema checks
- replay and compaction outcomes preserve truthful liveness state

### PR D

- the scenario pack is understandable and reproducible
- the pack includes a mutating replay-safety lane, not only read-only flows
- reports are readable by humans and automation
- parity claims are evidence-backed, not anecdotal

Expected artifacts from PR D:

- `qa-suite-report.md` / `qa-suite-summary.json` for each mode

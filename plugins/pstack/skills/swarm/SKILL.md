---
name: swarm
description: "Fan out N parallel workers, drain them, and return one report. Use for /swarm, 'swarm this', or parallel coverage, races, gauntlets, and exploration."
---

# Swarm

On Codex, read the [platform mapping](../poteto-mode/references/codex-tools.md), including its per-skill notes, before following this skill.

On GitHub Copilot, read the [platform mapping](../poteto-mode/references/copilot-tools.md), including its per-skill notes, before following this skill.

Fan out N parallel workers. They may cover separate slices, race the same brief, or mix both. The parent waits, aggregates, and returns one report.

## Start

Open a todolist with one entry per phase before launching anything.

1. Frame
2. Fan out
3. Aggregate
4. Report

## Phase A: Frame

1. State the done predicate and the artifact or report the swarm must return.
2. Choose the shape. Partition into slices, race N workers on identical briefs, or mix both. For a race or mixed shape, declare `first pass`, `rank all`, or `best-of` before spawning.
3. Set N from the user or derive it from the shape. N is total workers, not the number that run at once.
4. Pick the worker model from the `swarm workers` line in `pstack-models.md`. If the sheet or that line is missing, use the default in [Models](#models). Resolve each worker through [provider dispatch](../poteto-mode/references/provider-dispatch.md) before native mapping or fallback. For native `auto` or `inherit-parent`, omit `model` so the workers run on the parent model. If the native `Agent` tool rejects a model slug, use the default and say so. If it rejects the default, use the closest valid slug of the same family from its error message. External failures and capacity errors do not authorize that fallback. For a model race, name each arm's model up front.
5. Give each worker its own writable output when it writes. When workers verify or measure commits, each brief names the exact SHAs. A measurement brief also names the method (sample count, what one sample is, order). The worker records both in its result.

## Phase B: Fan out

Launch the N workers under the caller's scheduling policy. Native workers use one message with `subagent_type: "general-purpose"`, `run_in_background: true`, and the step 4 model, left unset for `auto` or `inherit-parent`. External workers use provider dispatch directly. Claude Code subagents all run on this machine, so isolation comes from the worktree or output directory assigned in Phase A, not from a remote environment.

Assign each external writer a caller-created linked worktree. Keep a worker table with its slice or race arm, model, effort, workspace, and native task or unique external prompt and record. Ask external writers to edit, verify, and report. The caller collects their diffs, verifies them, and commits them before any integration phase requires worker SHAs. Retain the worker's test and source-revision evidence alongside the caller-created commit SHA.

When a worker must start from a non-default branch, check that branch out in the worker's own worktree and name the worktree path in its brief.

Every brief stands alone. Include the goal, scope, exact slice or race arm, how to verify, and what to report. Reports use `PASS`, `ISSUES`, or `BLOCKED` with evidence. A worker that can prove a defect reports `ISSUES` and lists every issue it can prove, not only the first.

If a worker drops out, proceed with N-1 and note it. An external code-8 observation keeps the worker pending, and a failed external run is a gap without native substitution. Inspect partial edits in a failed writer's isolated worktree before deciding whether to collect them. Only a verified completed result can satisfy coverage or the race's pass rule.

## Phase C: Aggregate

Read the terminal results. Drop a result that does not record the SHAs and method its brief names, and launch that worker once more on the same target. For an external retry, use a new prompt and record. After a second miss, record a gap. A gap does not count as a pass. For coverage, every required slice needs a result. For a race, apply the selection rule declared up front. Use first pass, rank all, or best-of. Keep observation, cancellation of losing arms, and fan-in in the caller. Do not paste raw worker dumps.

Keep a compact result table, one-line evidenced issues, and explicit gaps or dropouts.

## Phase D: Report

Return one consolidated in-chat report with the table, issue one-liners, gaps or dropouts, and the race rule when used.

## Models

Role defaults, stamped from `plugins/pstack/models.json` (edit there, rerun `tools/generate.mjs`). A matching role line in the `pstack-models.md` override sheet overrides each at runtime; `/setup-pstack` writes it and lists its path per runtime. Resolve explicit external choices through [provider dispatch](../poteto-mode/references/provider-dispatch.md) before native model mapping or fallback.

- swarm workers: `opus`

## Reasoning effort

Resolve external seats through [provider dispatch](../poteto-mode/references/provider-dispatch.md) before native effort mapping. An `xai:<exact-id>` entry resolves effort from its `@<effort>` suffix or an explicit non-session sheet default. Validate that concrete effort against the exact provider model and pass it explicitly to relay. Missing or `session` effort blocks the seat. Do not inherit a native session effort, use native effort agents, or convert `max` to `xhigh` for it.

A native role value in the override sheet may name a reasoning effort after its model, as in `opus @xhigh`. Native levels on Claude Code: `low`, `medium`, `high`, `xhigh`, `max`. Which ones apply depends on the model. A value without `@` takes the sheet's `default effort` line, a level or `session`, and `session` when the sheet has no such line. `session` sets no effort, so the dispatch is the usual one. Strip the suffix before reading the model: `inherit-parent` or `auto` still omits `model` at every level, and a native model name is passed as `model`. On Claude Code, a native level picks the effort agent from the `subagent_type` you would otherwise use. `pstack:poteto-agent` becomes `subagent_type: "pstack:poteto-agent-<level>"`. `general-purpose`, or no `subagent_type`, becomes `subagent_type: "pstack:effort-<level>"`. The effort agents set only `effort`, so the model you pass still decides the model. On Codex, pass the native level as `spawn_agent`'s `reasoning_effort` and keep the usual instructions.

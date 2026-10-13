---
name: arena
description: "Spawn N parallel candidates at the same task, pick a base, graft the strongest parts of the losers into it. Use for /arena, 'arena this', 'throw it in the arena', or when one attempt at a non-trivial artifact would lock in the wrong shape."
---

# Arena

On Codex, read the [platform mapping](../poteto-mode/references/codex-tools.md), including its per-skill notes, before following this skill.

On GitHub Copilot, read the [platform mapping](../poteto-mode/references/copilot-tools.md), including its per-skill notes, before following this skill.

Open a todolist with one entry per phase (Frame, Fan out, Cross-judge, Pick, Graft, Verify) before launching anything.

## Phase A: Frame

Every candidate gets the same prompt, so the prompt is the contract.

1. State the artifact each candidate is producing.
2. Derive the rubric, 3-6 concrete gradeable criteria for what success looks like on *this* task. It is the picker's tool in Phase D. Candidates only see the task.
3. Pick the runners. Use the `arena runners` line in `pstack-models.md`. If the sheet or that line is missing, run one each on the defaults in [Models](#models). Resolve each seat through [provider dispatch](../poteto-mode/references/provider-dispatch.md) before native mapping or fallback. On the native branch, an `auto` or `inherit-parent` entry in this line or the cross-judge line means the parent model, so omit `model` for it. If the native `Agent` tool rejects a configured model, run that seat on its family's default and say so. Families go by model name, such as Opus, Fable, or Sonnet. With no family match, use the single-role default in [Models](#models). If it rejects a default, use the closest valid slug of the same family from its error message. Spawn more runners when the arena covers multiple design directions. Use the same model N times when the work is generation-bound rather than judgment-sensitive.
4. Give each candidate its own output location (a git worktree where possible, otherwise `/tmp/arena-<slug>/candidate-<n>/`), per the **separate-before-serializing-shared-state** principle skill.

## Phase B: Fan out

Launch all N seats with the same completed task and grounding. Native seats use subagents in one message with `run_in_background: true`. External seats use provider dispatch directly. Give each seat its own output path and instructions to produce the artifact and a short rationale naming the alternatives it considered and what it rejected.

Keep a candidate table with each label, model, effort, output path, and native task or external record. Save a completed read-only external answer into its candidate directory. For an external writer, collect and verify its isolated worktree and create any commit the artifact requires. A code-8 running result remains pending. A failed external run is a dropout without native substitution, and partial artifacts do not enter judging.

If a candidate produces no output, proceed with N-1 and note the dropout.

## Phase C: Cross-judge

After every Phase B seat has a completed result or recorded dropout, choose one model from the `arena cross-judge pool` line in `pstack-models.md`. If the sheet or that line is missing, choose from the runner defaults in [Models](#models). Prefer a different model family from the parent's. Resolve the judge through provider dispatch before native mapping or fallback. A native judge uses one readonly subagent, with the host's actual access controls. An external judge uses read-only relay dispatch and its own seat-to-record entry. It sees the rubric and completed candidates by path label, scores each criterion, and recommends a base with rationale. It runs in parallel with your reading in Phase D. Don't launch it while an author is pending or writing.

Wait for the judge's completed answer before comparing verdicts in Phase D. A running observation does not release that barrier. If the judge fails, record the missing independent verdict and keep any required review gate closed. Do not replace an external judge with a native model.

## Phase D: Pick a base

Read every candidate end to end before picking. Score each against the rubric criterion by criterion, not on holistic feel, and compare with the cross-judge. Agreement on the base confirms the pick. Disagreement means one of you is biased or the rubric was ambiguous. Read both rationales before deciding.

Pick the base a future maintainer can extend most easily without breaking invariants. When two feel tied, prefer the cleaner boundary or smaller API, per the Laziness Protocol.

## Phase E: Graft

Walk each losing candidate once more for what is worth porting into the base, usually one or two things per candidate, not most of it. Fold each graft in by hand, per the **redesign-from-first-principles** principle skill, so the result stays coherent under one mental model. Don't paste mechanically.

If the candidates converge on one shape, that is strong agreement. Note it and ship the consensus shape. No graft is needed. If they wildly diverge, Phase A was under-specified. Reframe and re-run rather than averaging the divergence.

## Phase F: Verify

Verify the synthesized artifact like any other output, per the **prove-it-works** principle skill. If verification finds a problem the arena missed, either Phase A was wrong (re-frame and re-run) or a candidate caught it and you missed the graft (go back to Phase E). Don't paper over it.

## Outputs

One synthesized artifact, with a short synthesis note alongside naming the base and why, the cross-judge's verdict, each graft with its source candidate, what was rejected and why, any convergence or dropouts, and the verification result.

## Models

Role defaults, stamped from `plugins/pstack/models.json` (edit there, rerun `tools/generate.mjs`). A matching role line in the `pstack-models.md` override sheet overrides each at runtime; `/setup-pstack` writes it and lists its path per runtime. Resolve explicit external choices through [provider dispatch](../poteto-mode/references/provider-dispatch.md) before native model mapping or fallback.

- arena runners: `opus`, `fable`, `sonnet`, `haiku`
- arena cross-judge pool: `opus`, `fable`, `sonnet`, `haiku`

## Reasoning effort

Resolve external seats through [provider dispatch](../poteto-mode/references/provider-dispatch.md) before native effort mapping. An `xai:<exact-id>` entry resolves effort from its `@<effort>` suffix or an explicit non-session sheet default. Validate that concrete effort against the exact provider model and pass it explicitly to relay. Missing or `session` effort blocks the seat. Do not inherit a native session effort, use native effort agents, or convert `max` to `xhigh` for it.

A native role value in the override sheet may name a reasoning effort after its model, as in `opus @xhigh`. Native levels on Claude Code: `low`, `medium`, `high`, `xhigh`, `max`. Which ones apply depends on the model. A value without `@` takes the sheet's `default effort` line, a level or `session`, and `session` when the sheet has no such line. `session` sets no effort, so the dispatch is the usual one. Strip the suffix before reading the model: `inherit-parent` or `auto` still omits `model` at every level, and a native model name is passed as `model`. On Claude Code, a native level picks the effort agent from the `subagent_type` you would otherwise use. `pstack:poteto-agent` becomes `subagent_type: "pstack:poteto-agent-<level>"`. `general-purpose`, or no `subagent_type`, becomes `subagent_type: "pstack:effort-<level>"`. The effort agents set only `effort`, so the model you pass still decides the model. On Codex, pass the native level as `spawn_agent`'s `reasoning_effort` and keep the usual instructions.

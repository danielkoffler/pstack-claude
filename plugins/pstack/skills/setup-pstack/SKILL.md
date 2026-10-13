---
name: setup-pstack
description: Configure which models pstack uses per role. Detects available models and writes the current runtime's override sheet. Use for /setup-pstack, "configure pstack models", changing pstack's model choices, or turning the SessionStart hook on or off.
---

# Setup pstack

On Codex, read the [platform mapping](../poteto-mode/references/codex-tools.md), including its per-skill notes, before following this skill.

On GitHub Copilot, read the [platform mapping](../poteto-mode/references/copilot-tools.md), including its per-skill notes, before following this skill.

On GitHub Copilot, follow [the Copilot setup](copilot.md) in place of steps 1, 3, 4, and 7 and step 6's header.

On another runtime, read [Other runtimes](#other-runtimes) below for where the sheet lives and how it loads; the steps are the same.

Write the current runtime's per-role model override sheet, using the path in [Other runtimes](#other-runtimes). Each pstack skill names a default model inline; the override sheet adapts those defaults to the models you actually have access to.

Claude Code has no auto-applied "rules" mechanism like Cursor's `.mdc`. The Claude Code config directory is `$CLAUDE_CONFIG_DIR` when that variable is set and `~/.claude` otherwise. This skill calls it `<config>`. Inclusion is explicit: the user adds a line to `<config>/CLAUDE.md` (or their project `CLAUDE.md`) such as:

```text
@<config>/pstack-models.md
```

with `<config>` written as the resolved path, so the file is loaded as context for every session.

The Codex home directory is `$CODEX_HOME` when that variable is set and `~/.codex` otherwise. This skill calls it `<codex-home>`.

## Steps

### 1. Detect available models

Enumerate the model names the `Agent` tool's `model` parameter accepts in this session. That is the dependable source. On Claude Code they are the family names listed in [Models](#models) below, each running that family's current model, and a full model ID is rejected. The default panel is listed there too. The panel is chosen for cross-family diversity. Ask the user to confirm or paste any additional slugs they want available. Never write a real slug you have not confirmed is available. The aliases `inherit-parent` and `auto` are always valid even though they are not detected slugs. Both mean the role runs on the parent session's model, which the `Agent` call expresses by omitting `model`.

On Claude Code and Codex, an explicitly requested external seat follows [provider dispatch](../poteto-mode/references/provider-dispatch.md). Load the separately installed `agent-relay` skill and run its read-only `xai.mjs catalog` command. Keep those exact IDs and supported efforts separate from the native model set. Discovery does not prove account entitlement. If discovery or the required relay contract is unavailable, mark the external choice as blocked. Do not offer Grok aliases, a guessed ID, or a native replacement.

### 2. Load current state

The default role-to-model mapping is the rule shape shown in the Write the override sheet step below. If the current runtime's sheet already exists, read it and treat its values as the current choices. Otherwise start from those defaults. A line whose role is not in that shape, such as `how critics`, is from a retired role. Drop it. An older sheet may name full model IDs that start with `claude-`, which the `Agent` tool rejects. Replace each with its family name, the word after `claude-`.

### 3. Map and confirm

Show every role with its current model, marking any native slug not in the detected set or external entry not validated through provider dispatch as needing a choice. Also list each line step 2 dropped or rewrote. Ask whether to accept as-is or change specific roles, offering the detected native models plus `inherit-parent` and `auto` as the options. Offer validated external IDs only when the user requests external seats in workflows with a caller adapter. Prefer `AskUserQuestion` over free text. For panel roles (arena runners, architect runners, interrogate reviewers) the value is a list, and one seat runs per entry, alias entries included, so the list length sets the count. `arena cross-judge pool` is also a list, but Arena selects one value from it whose model family differs from the parent's when possible. `swarm workers` is the default model for every worker unless a race or comparison assigns another model per arm.

Then ask for the default reasoning effort, the `default effort` line. It is `session`, which keeps a native role's parent session effort, or one of the levels in [Models](#models). Start from the default listed there. Every native role value without a suffix runs at it. Then ask whether any role should run at another level. On Claude Code a native role value may carry one after its slug, as in `<slug> @xhigh`; panel entries take their own, as in `<slug> @xhigh, <slug> @max`. Each native level dispatches through the plugin's effort agent of that level, so the choice holds without changing the session. Leave the suffix off for the default effort. Each external entry resolves effort from its suffix or an explicit non-session sheet default. Validate that value against its exact provider model and pass it explicitly to relay. External sessions do not inherit native `session` effort or use native effort conversion.

### 4. Choose whether the session hook routes tasks

On Claude Code and Codex, the plugin's `SessionStart` hook injects the poteto-mode mandate on startup, resume, clear, and compact. Codex asks the user to trust plugin hooks through `/hooks` before running them. On Pi, the pstack extension adds the same mandate to the system prompt at every agent start. Ask whether to keep the hook. The default is on. The answer is the `session hook` line in the current runtime's sheet: `on` or `off`. With no sheet or no line, the hook injects. The line is inert on other runtimes.

### 5. Validate

Split the model token from its effort suffix and validate external entries before native aliases or levels. An `xai:<exact-id>` entry must match the relay catalog and name a role with a caller adapter on this runtime. Resolve its concrete effort from the suffix or an explicit non-session sheet default, then require that catalog to list the effort for that model. Reject missing effort, `session`, an unsupported level, and Grok aliases. Never map external `max` to `xhigh`, choose a provider default, or replace a rejected provider entry with a native slug. If the external choice is invalid, stop and ask for a supported explicit choice.

Every native real slug written must be in the detected set. `inherit-parent` and `auto` always pass. Validate the native slug without any `@<level>` suffix, and its level against the effort levels in [Models](#models). The `default effort` value is one of those levels or `session`. On Codex, the native levels are the `reasoning_effort` values the selected Codex model accepts instead. If a chosen native slug or level is not available, stop and ask again.

### 6. Write the override sheet

Write the current runtime's sheet with the shape below. Overwrite the whole file so re-runs stay idempotent.

```markdown
# pstack model configuration

Per-role model overrides for pstack skills. Each pstack SKILL.md names its defaults in a Models section; the values here override those defaults. Delete a line to fall back to the skill default. A value of `inherit-parent` or `auto` runs that role on the parent session's model (the `Agent` call omits `model`); an alias entry in a panel list still counts toward that panel's fan-out. A native model may carry a reasoning effort, as in `opus @xhigh` (levels: low, medium, high, xhigh, max); on Claude Code the native role then runs through the pstack effort agent of that level, each entry of a panel list on its own. `default effort` sets the level for a value without a suffix; native `session` keeps the parent session's effort. An external `xai:<exact-id>` entry needs a workflow with a caller adapter and a concrete effort from its suffix or an explicit non-session sheet default. Validate that effort against the exact provider model and pass it explicitly to relay. Missing or `session` effort blocks the external seat. It uses the separately installed agent-relay skill instead of native aliases, effort agents, or fallback. `session hook: off` stops the Claude Code or Codex SessionStart hook, or the pstack Pi extension, from injecting the poteto-mode mandate; any other value, or no line, leaves it on.

feature, refactoring: opus
bug-fix: fable
perf-issue: fable
hillclimb: fable
judgment and prose: opus
strongest judgment: fable
how explorer: opus
how explainer: opus
why investigators: opus
why synthesizer: opus
reflect tooling: opus
reflect judgment, divergent, synthesizer: opus
arena runners: opus, fable, sonnet, haiku
arena cross-judge pool: opus, fable, sonnet, haiku
swarm workers: opus
architect runners: opus, fable, sonnet, haiku
interrogate reviewers: opus, fable, sonnet, haiku

default effort: session
session hook: on
```

### 7. Wire it in

On Claude Code, if `<config>/CLAUDE.md` does not already include `<config>/pstack-models.md`, append an `@` line naming the sheet's resolved path, such as `@~/.claude/pstack-models.md`, so the model rows load on every session. If the user prefers project scope, add the include to the project's `CLAUDE.md` instead.

On Codex, keep `<codex-home>/pstack-models.md` as the single source. If no existing instruction loads it, add a pointer to `<codex-home>/AGENTS.md` that tells the agent to read the resolved sheet path before running pstack. Preserve an existing pointer. Do not duplicate model rows, effort choices, or the hook setting in `AGENTS.md`. The plugin hook reads the setting directly from the sheet.

### 8. Confirm

Tell the user where the override was written, how its model rows load, and whether the plugin hook is on. Re-running this skill updates the override sheet.

### 9. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a project `verify` or `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /create-verification-skill." On yes, invoke [`/create-verification-skill`](../create-verification-skill/SKILL.md). On no, move on without pushing.

## Other runtimes

The role lines are the same everywhere. What differs is the sheet path, how the runtime loads it, and how you list models. Detect models with the runtime's own tool and never write a slug you have not seen listed. A runtime whose subagent call has no model parameter still gets the sheet, as the record of the user's choice, and applies it where it can. The `session hook` line applies to the Claude Code, Codex, and GitHub Copilot plugins and to the pstack Pi extension.

| Runtime | Sheet | Load | List models | Status |
| --- | --- | --- | --- | --- |
| Claude Code | `<config>/pstack-models.md` | `@<config>/pstack-models.md` in `<config>/CLAUDE.md` | the `Agent` tool's model parameter | verified live |
| Codex | `<codex-home>/pstack-models.md` | an instruction pointer in `<codex-home>/AGENTS.md` loads the single sheet; the plugin reads the hook setting | native Codex models and requested external capabilities, see [codex-tools.md](../poteto-mode/references/codex-tools.md#model-names) | native hook contract tested; discovery verified |
| Pi | `pstack-models.md` in the Pi agent directory, `$PI_CODING_AGENT_DIR` or `~/.pi/agent` | read by the pstack Pi extension, model rows and hook setting both; no include line | `pi --list-models`, see [pi-tools.md](../poteto-mode/references/pi-tools.md#model-names) and its `setup-pstack` note | extension contract tested offline; live results in the repository's `docs/pi-equivalence.md` |
| GitHub Copilot (CLI and app) | `${COPILOT_HOME:-~/.copilot}/pstack-models.md` | the plugin hook checks it and injects its role lines at session start; skills-only installs read it with `view` | the `task` tool's `model` enum, see [copilot-tools.md](../poteto-mode/references/copilot-tools.md#model-names) | hook contract tested; CLI install smoke-tested |
| opencode | `~/.config/opencode/pstack-models.md` | add the path to the `instructions` array in `opencode.json` | the `models` slash command in the session | from published docs, no live session |
| Gemini CLI | `~/.gemini/pstack-models.md` | `@~/.gemini/pstack-models.md` in `~/.gemini/GEMINI.md` | the `model` slash command in the session | from published docs, no live session |
| Prime Agent | no documented sheet path; Prime's configuration chooses models | | | no live session |

## Models

Stamped from `plugins/pstack/models.json` (edit there, rerun `tools/generate.mjs`).

- Available Claude models: `opus`, `fable`, `sonnet`, `haiku`
- Default panel: `opus`, `fable`, `sonnet`, `haiku`
- Reasoning effort levels: `low`, `medium`, `high`, `xhigh`, `max`
- Default reasoning effort: `session`
- Single-role default: `opus`

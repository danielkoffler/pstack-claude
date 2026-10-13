# Dispatch an external model seat

Use this branch for an explicitly selected `xai:<exact-id>` seat on Claude Code or Codex. The `architect`, `arena`, `interrogate`, and `swarm` workflows adapt their results as described in their own steps. Other workflows and runtimes need a caller adapter before using an external seat. If that adapter is absent, record a blocked seat.

The separately installed `agent-relay` skill supplies Grok execution. Load it by skill name and locate its scripts from the discovered skill directory. If it is unavailable or lacks the contract below, record the dependency failure. Do not vendor its scripts or replace the seat with a native model.

## Resolve the target before native dispatch

1. Read the calling workflow's role from its runtime's `pstack-models.md`. Keep its seat label in a caller-owned table.
2. Split each value into its model token and optional `@<effort>` suffix. Inspect the model token before native alias mapping, effort wrappers, or model-rejection fallback.
3. For `xai:` tokens, require the exact catalog ID. Resolve effort from the entry's `@<effort>` suffix, or from an explicit non-session `default effort` in the sheet when the suffix is absent. Use `agent-relay`'s read-only `xai.mjs catalog` command to validate that exact ID and concrete effort. Missing discovery, an unknown ID, missing or `session` effort, or an unsupported effort blocks the seat before launch. Always pass the validated effort explicitly to relay. Never select the newest match, use a provider default, or convert `max` to `xhigh`.
4. For native tokens, keep the host's native dispatch. `auto` and `inherit-parent` omit the native model override. Native session-effort inheritance, effort wrappers, and model-rejection fallback apply only on this branch. A host capacity error is not a model rejection. Reject Grok targets without the `xai:` prefix, including bare Grok IDs and aliases such as `grok`, `xai`, and `grok:<id>`, as external configuration errors instead of sending them to native fallback.

Setup validates capabilities, not account entitlement. A supported catalog entry can still fail at launch because of authentication, account access, or host permissions. Return that failure to the caller. A change of provider, model, effort, or billing route requires the caller's explicit authorization.

## Build one concrete request per seat

The caller owns the completed standalone prompt, access mode, workspace, deadline, silence policy, scheduling, retries, cancellation, and synthesis. External sessions do not inherit the parent's transcript or native agent instructions.

Give each independent run a unique absolute prompt filename. Include the task, accessible context paths or their contents, constraints, verification method, output requirements, and report shape. Give equivalent task context to native and external seats. Keep the prompt bytes unchanged while the run exists.

For read-only work, select `--mode read-only` and an explicit repository workspace. The caller saves a completed answer into that seat's output directory when the workflow requires files. For direct edits, create and assign a separate linked worktree before launch and select `--mode write`. Ask Grok to edit, verify, and report its changes. The caller inspects the worktree, runs required checks, and creates commits before a later phase requires worker SHAs. The relay creates no worktree and commits no changes.

Choose invocation metadata for `--owner`. Use `interactive` for a top-level caller, `subagent` for a caller already running as a native child, and `unattended` for an unattended caller. Do not create a native carrier agent solely to launch relay.

Invoke the discovered relay script with this command shape. Replace every placeholder with the caller's concrete value. Add a deadline or positive observation window only when the caller chooses one.

```bash
node <relay-scripts>/relay.mjs run --prompt-file <absolute-unique-prompt> --model xai:<exact-id> --effort <supported-effort> --workspace <absolute-workspace> --mode <read-only-or-write> --owner <invocation-owner> --silence-limit <nonnegative-seconds>
```

The required relay contract includes exact xAI model and explicit effort validation, `--silence-limit`, and positive `--window` values for every owner. Zero silence disables silence cancellation. An optional `--deadline` is relative seconds or a supported epoch value. Deadlines and silence limits require an active observer. There is no deadline timer between observation windows.

## Observe and collect the result

Associate the seat label with its prompt path, adjacent record path, returned provider handle, and assigned output or worktree. Retain the host execution session separately. A Bash task ID or Codex `session_id` is not the provider identity. Never reuse one prompt or record for another seat or retry.

On Claude Code, use background Bash for `run` and `watch`. A bounded foreground call needs a positive `--window` and a Bash timeout covering that window plus 30 seconds. On Codex, launch through `exec_command` or its current shell equivalent. Keep any returned `session_id` and use `write_stdin` with empty input to collect its output. Use the host's supported permission path when needed.

These are separate operations on the saved record. Run `watch` when another observation is needed. Run `cancel` only after the caller decides to stop the job.

```bash
node <relay-scripts>/relay.mjs report <record-path>
node <relay-scripts>/relay.mjs watch <record-path>
node <relay-scripts>/relay.mjs cancel <record-path>
```

`report` reads persisted evidence. It does not refresh liveness. A positive `--window` bounds observation on `run` or `watch`. Exit code 8 with `outcome: running` leaves the job running. Keep the seat pending and observe again or cancel according to the workflow. Do not relaunch it, count it as a result, advance a completion barrier, or substitute a model.

Accept a completed answer only when exit code 0 and a valid report identify `outcome: completed`, an `answerFile`, and its `answerSha256`. Read the answer file and verify its digest against that hash. Retain the unchanged report alongside the answer. Preserve requested, resolved, and served model and effort evidence. Unknown served identity stays unknown. Provider completion does not establish that a candidate meets its rubric or that a review is correct.

A terminal non-completion is a dropout or gap under the caller's workflow. A partial answer is not a completed result. A command error, including code 64, is distinct from a provider outcome and need not have a record. Surface both without native substitution. Retries get new prompt and record paths and retain the requested target unless the caller explicitly authorizes a change.

For a write run, inspect workspace status and partial edits even when the run fails or stops. `stop.confirmed` describes absence of the recorded local process group, not remote compute or all possible detached writers. Exclude failed candidate artifacts from judging, and establish that the assigned workspace is safe to collect before committing or integrating edits. Native Stop is not a relay cancellation contract.

Keep orchestration in the calling workflow. It selects seats, observes all pending runs, decides retries and cancellations, applies barriers, judges answers, integrates changes, and reports gaps. Relay returns execution evidence for one job.

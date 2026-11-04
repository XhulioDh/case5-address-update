# Petri feasibility check

**Question:** can a Petri audit route the target's tool calls to our own local
Python functions (`portal_sim.py`), so that the *simulator* — not the auditor
model — decides each result and records the state change?

**Short answer:** not with Petri's standard target. Its standard design has the
auditor model write every tool result. Petri does provide an official place to
plug in your own target loop (`audit(target=...)`), and that loop could call
`portal_sim.py` directly. That route looks feasible from reading the source,
but **nobody has run it**. It is marked **unverified** below. The offline
simulator does not depend on it.

## What was inspected, and what was not

| Item | Value |
|---|---|
| Package | `inspect-petri` **3.1.1** (the version your laptop reported) |
| Source 1 | PyPI sdist `inspect_petri-3.1.1.tar.gz`, SHA-256 `d9f96652b5736ac6f26c45efb65d3e4be48ee95e1cd30f03efbdf199970d6bc9`, uploaded 2026-10-01 |
| Source 2 | GitHub `meridianlabs-ai/inspect_petri`, tag `3.1.1` → commit `1443ed5ffcd5ffa0da923f61504ca842dd07378d` ("chore(main): release 3.1.1 (#150)", 2026-10-01) |
| Match | `src/inspect_petri/` in the tag and the sdist are identical, apart from the build-generated `_version.py` |
| Method | **Source reading only.** Nothing was installed or run, and no model was called. |
| Not inspected | Your Windows laptop. We cannot confirm that your installed copy matches this source. Running `pip show inspect-petri` there would confirm the version. Also not inspected: `inspect_ai` internals, and the Petri Dish extension (a separate package). |

## Findings from reading the source (tag 3.1.1)

1. **The standard target gets its tool results from the auditor.**
   `src/inspect_petri/target/_agent.py:74-78`: when the target makes tool calls,
   it sends its output to the auditor (`context.send_output`). It then waits for
   `context.tool_results(tool_calls)`, and those results are whatever the auditor
   staged. The auditor stages them with `send_tool_call_result`
   (`src/inspect_petri/tools/_toolcalls.py:279`). That tool's own instructions
   ask the auditor model to *invent* realistic output and keep the state
   consistent itself.
   The docs say so directly: `docs/using/concepts.qmd:21` — "tools the auditor
   declares and whose results it fabricates — no real code runs".

2. **`target_tools="fixed"` does not change who writes the results.**
   `docs/components/custom_task.qmd:77`: in fixed mode the tools are
   pre-registered from `metadata["tools"]` (`src/inspect_petri/_auditor/auditor.py:257`,
   `_read_tools`), but "the auditor still responds to tool calls with
   `send_tool_call_result()`". Fixed mode only fixes the tool *list*.

3. **A custom target agent is officially supported.**
   `src/inspect_petri/_task/audit.py:36` and `:93` — `audit(target=...)`
   ("Custom target agent driver … Supply this to audit a target with custom
   scaffolding"). The target is a function `(state, context)` called by
   `src/inspect_petri/_auditor/auditor.py:143-190` (`_run_target`,
   `_run_trajectory`, `_call_target`). It receives a `TargetContext`
   (`src/inspect_petri/target/_context.py`), which offers `context.tools`,
   `send_output`, `user_message`, `tool_results` and `replayable`.
   Nothing in the source we read forces a custom target to call
   `context.tool_results()` for every tool call.

4. **Precedent for tools that run natively.** `docs/extensions/petri-dish.qmd:18`
   says that in Petri Dish, "Passthrough tools (planning, bookkeeping) execute
   natively instead" of being simulated by the auditor. So local execution
   inside a Petri-managed target is a pattern the authors use. We did not
   inspect Petri Dish's code.

5. **Rollback is a problem for real state.** If the auditor rolls back
   (`enable_rollback=True` is the default, `_task/audit.py:26`), Petri starts a
   new trajectory and *replays* recorded steps
   (`_auditor/auditor.py:_run_trajectory`; `target/_context.py:126`, `replayable`).
   A local simulator's state would not roll back with it, so the action log
   could contain events from a branch the transcript no longer shows. The
   simplest fix is to turn rollback off (`enable_rollback=False`) for this pilot.

6. **Hosted tools are blocked.** `target/_synthetic.py:16,27` strips provider
   "server-side" tool options and aborts the run if a provider executes a tool
   itself. That does not affect locally run Python functions.

## Possible integration (sketch only — not written as code, not run)

A custom target that would:

1. Read `TOOLS` from `portal_sim.py` and pass them to the model as tool definitions.
2. After each model generation, for each tool call: run
   `portal.call_tool(tc.function, tc.arguments, tool_call_id=tc.id)` and append
   the result as a tool message. This happens locally, and the auditor is not
   asked for a result.
3. When the model replies with no tool calls (a question or a completion
   message), send that output to the auditor and wait for the citizen's reply
   through `context.user_message()`.
4. Run with `enable_rollback=False`, and probably `target_tools="none"`,
   so the auditor cannot add tools or fabricate tool results.

## What remains unresolved

| # | Unresolved point | Why it matters |
|---|---|---|
| U1 | The custom-target route has never been run. | Unknown failure modes in the protocol (e.g. `expect()` / `send_output` sequencing when tool calls never reach the auditor). |
| U2 | **Transcript ↔ action-log matching.** `portal_sim` records `tool_call_id`, `last_message_id` and `last_event_id`. Whether Petri's saved transcript exposes the same tool-call ids and message order has not been checked against a real log. | This determines whether reviewers can line up portal events with messages. |
| U3 | **Who plays the citizen.** In Petri, the auditor *model* writes the user messages. A seed that tells it to reply "No, keep my letters going to 6 Cedar Lane" is a request, not something that is enforced. It could paraphrase, volunteer the preference, or skip the follow-up. | The citizen script would need checking in every transcript. Another option is to have the custom target supply scripted replies itself (not designed yet). |
| U4 | **Deciding which message is the completion message.** Nothing has been decided. | The snapshot must be taken at the right message. |
| U5 | Exact behaviour of `inspect_ai` (≥ 0.3.236, the version Petri requires) when building tool messages. Not inspected. | Needed to write the custom target correctly. |
| U6 | Whether your laptop's installed copy matches tag 3.1.1. | Source inspection was done in a cloud container, not on your laptop. |

**Important:** a seed that asks the auditor to *narrate* a rejection of the
staff-only tool is **not** an enforced rejection. The rejection only counts
when `portal_sim.py` refuses the write and logs it (`write_rejected` event,
record unchanged). Under the standard Petri target, that guarantee does not exist.

**Decision for now:** go ahead with the standalone simulator. Integration
work is optional and later. It is TBD whether to attempt it at all.

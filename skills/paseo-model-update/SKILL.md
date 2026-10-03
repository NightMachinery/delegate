---
name: paseo-model-update
description: Report or change the model and reasoning effort of the agent you are, when you run inside Paseo. Use it whenever the user asks what you are running ("what model/effort are you on?", "verify you are Opus 5.5 xhigh", "did the switch take effect?") or asks you to switch yourself ("switch yourself to GPT 6.1 Sol high", "use Opus low from now on", "after the plan is approved, change to X", "can you change your own model?"), including requests gated on a later event. Status questions get a read-only answer that separates recorded settings, observed runtime and pending changes. Switches get a read-only preflight, honor the gate, apply a same-provider change with update_agent and verify it, and turn a cross-provider request into an explicit handoff instead of pretending update_agent can change providers.
---

# Paseo model update

Two jobs: saying truthfully what model and effort you are running (the
status-only section below), and changing them.

A Paseo agent can change its own model and effort only **within its provider
entry**. `update_agent` has no provider field: it hands the model ID to the
existing Claude or Codex session. Moving from Claude to Codex, or between two
entries of the same vendor (for example a personal and a work account), needs
a new agent. This skill makes that distinction early, says it plainly, and
changes nothing until the user's gate is met.

Read the installed official `paseo` skill for current tool and CLI syntax. The
behavior below was read from the Paseo 0.10.2 daemon sources; the file paths
and the reasoning behind each rule are in
[references/paseo-internals.md](references/paseo-internals.md). What matters
is the daemon your agent runs on, not the CLI on your PATH, and the two can
differ: read `daemonVersion` from `paseo status --json`, passing the same
`--host` or `--home` (or `PASEO_HOME`) that the session uses. When it is not
0.10.2, recheck the facts you rely on there.

## Rules that do not bend

- The user's explicit model, effort, account and timing requirements are
  binding. Do not substitute a "close" model (`gpt-6-sol` for `gpt-6.1-sol`),
  a different effort, or another account. If the exact target is unavailable,
  say so and ask.
- Never pass a model ID from another provider to `update_agent`. The daemon
  stores whatever string it receives without checking it, so a Claude agent
  told to use `gpt-6.1-sol` accepts the call and fails later, possibly
  leaving the conversation unusable.
- Do not change settings or launch agents to find out whether something
  would work. The preflight is read-only: status, provider lists, model
  lists, `inspect_provider`.
- Honor the gate the user set, and until it is met, preflight and report
  only. A gate that rests on the user's judgment ("after the plan is
  finalized", "when I say go") is met only by their explicit approval, never
  by your own view that the plan looks done. An objective gate the user
  specified ("after the tests pass", "once f2a7 is pushed") is met when you
  can verify the condition; cite that evidence when you act.

## Status only: what am I running?

For "what model and effort are you running?", "verify you are X" or "did my
change take effect?", answer read-only. Do not call `update_agent` or
`create_agent`, and do not start a handoff, even when the answer shows a
mismatch: report it, and offer the switch workflow only as an option.

1. Run the identity guard in step 1 below. Without a verified Paseo identity
   the recorded layer is unknown; native evidence may still answer the
   observed layer.
2. Report three layers, each with its source, because they can disagree:
   - **Recorded:** what Paseo has configured, from your MCP status: `model`,
     `thinkingOptionId`, `effectiveThinkingOptionId`, mode. This is what the
     next turn will use. It echoes requests, so it is not runtime proof.
   - **Observed:** what is running now, from runtime evidence (commands in
     the reference file). Claude's model is the `model` of the latest
     assistant message in your transcript; its effort is the `--effort` of
     your process, used only after the ancestry and session check passes.
     Codex records both in the latest `turn_context`.
   - **Pending:** every field where recorded and observed differ. Say when it
     applies: effort from the next turn on both providers, a Codex model from
     the next turn, a Claude model from its next request.
3. Say **unknown** for any field without evidence. Do not fill it from a
   product label ("Opus 5.5 (1M context)"), the user's description of you,
   your parent's model, a marketing alias such as "opus" or "latest", or your
   own sense of which model you are, including a model name in your
   instructions: those are claims from launch time, not observations.

## 1. Identify yourself exactly

1. Read `PASEO_AGENT_ID`. If it is empty, your identity is unavailable. That
   does not prove you are outside Paseo, since a tool that runs commands in
   a clean environment drops it too. Say the identity is unavailable and
   change nothing. For a status question, still report what native evidence
   shows; otherwise stop. Mention a native switch (`/model` in Claude Code or
   Codex) only when you know you run in a native interactive CLI the user
   is typing into, not under Paseo's SDK or app server.
2. If you are an in-process subagent of a Paseo agent, the environment
   names your parent, not you. Do not act on it; report to your parent.
3. Fetch your own record by that ID. Never pick "the agent with my title"
   from `list_agents`. The MCP tool and the CLI return different fields:
   - `get_agent_status` (MCP) has everything below: `provider`, `model`,
     `thinkingOptionId`, `effectiveThinkingOptionId`, `currentModeId`,
     `status`, `activeTurn`, `cwd`, `features`, and
     `persistence.sessionId`.
   - `paseo inspect "$PASEO_AGENT_ID" --json` has `Id`, `Provider`,
     `Model`, `Thinking` (the configured effort), `Mode`, `Status` and
     `Cwd`, but no session ID, effective effort, active turn or features.
4. Confirm the record is you: its cwd matches `PASEO_AGENT_CWD`, its status
   is `running` with an active turn, and `persistence.sessionId` matches your
   native session (`CLAUDE_CODE_SESSION_ID` for Claude, `CODEX_THREAD_ID`
   for Codex, when set). A mismatch means stop and report. With only the
   CLI, the session check cannot be made: report identity as partly
   verified, and before changing anything in place, get the MCP status or
   the user's confirmation that the ID is yours.
5. Record the current provider, model, configured and effective effort,
   mode, and feature values such as `fast_mode`. You need them to report,
   and to restore on a failed change.

If the user described you ("You are Opus 5.5 xhigh"), compare that with the
record and mention any difference.

## 2. Resolve the target, read-only

1. `list_providers` shows the provider entries and their status.
   `list_models` for a provider gives exact model IDs and each model's
   `thinkingOptions`.
2. Match the requested model by normalized ID or label (case, spaces,
   hyphens and dots ignored), and require exactly one match. Version numbers
   are part of the name: "6.1" never matches `gpt-6-sol`. Accept a later
   version only when the user said so ("or later").
3. Check the requested effort is in that model's `thinkingOptions`. Effort
   names repeat across providers (`high`, `xhigh`) without meaning the same
   budget, so never carry an effort across a provider change unless the user
   named it.
4. Find every **available** provider entry that serves the model. An entry
   binds an account through its configuration; its label proves nothing.
   - Your own entry serves it: the change can happen in place, on the
     account you already use.
   - Otherwise the target is another entry, and so possibly another
     account. Finding a candidate is not authorization to use it, even when
     it is the only one. Look for an account choice the user already made:
     in this request or conversation, in standing instructions, or under
     the account rules of the `delegate` skill when it is installed. If an
     authorized choice names one of the candidates, use it, however many
     candidates there are. If none does, list the candidates and ask; do not
     pick for the user.
5. Optionally call `inspect_provider` with the draft `provider/model`,
   effort and mode to see which modes and features the target supports.

## 3. Report the preflight

Report before changing anything, in this shape:

- **Current:** provider entry, model, effort, mode, and how you verified the
  record is yours.
- **Target:** provider entry, exact model ID, effort, and where each came
  from.
- **Path:** in place, or handoff, and why.
- **What changes:** timing, what context survives, permissions mode,
  account.
- **What I need from you:** the gate, plus any choice the request left open
  (an account not yet authorized).

When the user asked only "can you", stop here.

## 4a. Same provider entry: update in place

The conversation, its context and the permissions mode stay as they are.

1. At the gate, call `update_agent` once with only what is changing:
   `{agentId: $PASEO_AGENT_ID, settings: {model, thinkingOptionId}}`. Leave
   `modeId` and `features` alone unless the user asked. The daemon applies
   model before effort, which matters because Claude checks the effort
   against the model already set.
2. Keep it to that one call per turn. On Claude, an effort change marks the
   process for a restart at the next turn, and a later model change in the
   same turn performs that restart immediately, retiring the process that is
   running your turn.
3. The settings are applied one after another, not atomically. If the call
   fails, fetch your status again. When the model changed but the effort did
   not, an effort-only call is safe; a second model change waits for the
   next turn. Report either way. One correction, not a retry loop.
4. Fetch your status and compare `model` and `effectiveThinkingOptionId`
   with the target. This proves Paseo recorded the change; it does not prove
   the provider is running the new model. Claude's effort is echoed back as
   requested, not read from the runtime.
5. Tell the user the change applies **from the next turn**. An effort change
   during a turn is deferred by both providers (the MCP tool drops the notice
   saying so), and Codex reads the model at the start of each turn. Claude
   forwards the model to its live process at once, but do not claim any part
   of the current reply came from the new model.
6. A model the new target does not support for fast mode turns `fast_mode`
   off. Mention it if it was on.

After the next turn has started, runtime evidence is available if the user
wants proof (commands in the reference file): Claude's next assistant
messages in the transcript carry the model, and after an effort change its
restarted process carries the new `--effort`, which you may present only
after confirming that process serves your session; Codex writes a
`turn_context` record with `model` and `effort` for every turn.

## 4b. Different provider entry: hand off

You cannot become the other model. What you can offer is a new agent on the
target that continues the work from a briefing you write. Say so, together
with what is lost: your context beyond the briefing and your session's
history as working memory. The new agent gets the automatic-review
permission mode unless the user named another one (see below), whatever mode
you run in.

At the gate, follow the installed `paseo-handoff` skill, with these
additions:

- Use the exact `provider/model` resolved above and pass
  `settings.thinkingOptionId`.
- Unless the user named another mode, pass the mode in which permission
  prompts are reviewed automatically: `settings.modeId: "auto"` on Claude
  (Auto mode, a classifier reviews prompts) and `"auto-review"` on Codex (an
  auto-reviewer handles eligible approvals). This is the user's standing
  default. Choose by behavior, not by ID: Codex's `auto` is Default
  Permissions, which asks the user. Pass it explicitly rather than relying on
  the entry's own default. If the target has no such mode, say so and ask.
- The receiving agent is on another account or vendor. Before sending the
  briefing, apply the account and data-access check of the `delegate` skill
  when it is installed: everything the new agent can reach is exposed to that
  account, not only the briefing.
- Carry the user's binding decisions and the gate history into the briefing,
  so the new agent does not reopen settled questions.
- After the launch, stop working on the task yourself. Two writers on one
  task is the failure this step exists to avoid. Give the user the new
  agent's ID; it remains your subagent until they detach it, and you will be
  notified when it finishes.
- Verify the launch with `get_agent_status` on the new ID: provider, model
  and effort.

## Example

> You are Opus 5.5 xhigh. First plan with me. After the plan is finalized,
> using the paseo skill, switch yourself to GPT 6.1 Sol high. First check if
> you can do that.

Preflight now: your record says `claude`, `claude-opus-5-5`, `xhigh`, so the
description holds. `gpt-6.1-sol` with `high` exists only under Codex entries,
so this is a handoff, not an update. Naming the model chose the vendor, not
the account: use a Codex entry only if the user has already authorized that
account, and otherwise ask about it in the preflight. Report that you cannot
switch in place and what the handoff will carry, then continue planning.
"Finalized" rests on the user's judgment, so launch the handoff only when
they explicitly approve the plan.

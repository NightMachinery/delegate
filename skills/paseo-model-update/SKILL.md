---
name: paseo-model-update
description: Change the model or reasoning effort of the agent you are, when you run inside Paseo, or check whether you can. Use it whenever the user asks you to switch yourself to another model or effort ("switch yourself to GPT 6.1 Sol high", "use Opus low from now on", "after the plan is approved, change to X", "can you change your own model?"), including requests gated on a later event. Runs a read-only preflight, honors the gate, applies a same-provider change with update_agent and verifies it, and turns a cross-provider request into an explicit handoff instead of pretending update_agent can change providers.
---

# Paseo model update

A Paseo agent can change its own model and effort only **within its provider
entry**. `update_agent` has no provider field: it hands the model ID to the
existing Claude or Codex session. Moving from Claude to Codex, or between two
entries of the same vendor (for example a personal and a work account), needs
a new agent. This skill makes that distinction early, says it plainly, and
changes nothing until the user's gate is met.

Read the installed official `paseo` skill for current tool and CLI syntax. The
behavior below was read from the Paseo 0.10.2 daemon sources; the file paths
and the reasoning behind each rule are in
[references/paseo-internals.md](references/paseo-internals.md). If `paseo
--version` reports another version, recheck the facts you rely on there.

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
- A gate such as "after the plan is finalized" is met by the user's explicit
  approval, not by your own judgment that the plan looks done. Until then,
  preflight and report only.

## 1. Identify yourself exactly

1. Read `PASEO_AGENT_ID`. If it is empty you are not a Paseo agent: say so,
   and point to the native switch instead (`/model` in Claude Code or Codex).
2. Fetch your own record with `get_agent_status` (MCP) or
   `paseo inspect "$PASEO_AGENT_ID" --json`. Never pick "the agent with my
   title" from `list_agents`.
3. Confirm the record is you: its `cwd` matches `PASEO_AGENT_CWD`, its status
   is `running` with an active turn, and `persistence.sessionId` matches your
   native session (`CLAUDE_CODE_SESSION_ID` for Claude, `CODEX_THREAD_ID`
   for Codex, when set). A mismatch means stop and report.
4. Record the current `provider`, `model`, `thinkingOptionId` and
   `effectiveThinkingOptionId`, `currentModeId`, and feature values such as
   `fast_mode`. You need them to report, and to restore on a failed change.

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
   - Your own entry serves it: the change can happen in place.
   - Exactly one other entry serves it: that is the handoff target.
   - Several entries serve it (for example `codex` and an imported Codex
     profile): list them and let the user choose. Do not pick one for them.
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
  (provider entry, permissions mode for a new agent).

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
restarted process carries the new `--effort`; Codex writes a `turn_context`
record with `model` and `effort` for every turn.

## 4b. Different provider entry: hand off

You cannot become the other model. What you can offer is a new agent on the
target that continues the work from a briefing you write. Say so, together
with what is lost: your context beyond the briefing, your permissions mode
(a new agent starts in its entry's default mode unless `modeId` is given), and
your session's history as working memory.

At the gate, follow the installed `paseo-handoff` skill, with these
additions:

- Use the exact `provider/model` resolved above and pass
  `settings.thinkingOptionId`. Pass `settings.modeId` only as the user chose
  it or approved it in the preflight.
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
so this is a handoff, not an update. If both `codex` and an imported Codex
profile are available, ask which. Report that you cannot switch in place and
what the handoff will carry, then continue planning. Launch the handoff only
when the user approves the plan.

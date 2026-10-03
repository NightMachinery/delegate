# Paseo internals behind the rules

Read from the Paseo 0.10.2 daemon, installed with the CLI under
`$(npm root -g)/@getpaseo/cli/node_modules/@getpaseo/server/dist/server/server/`
(`S` below). `paseo status` prints `daemonVersion`; when it is not 0.10.2,
grep for the function names rather than trusting the line numbers.

## What `update_agent` does

`S/agent/tools/paseo-tools.js:1608` registers the tool. Its settings schema
(`:570`) accepts `modeId`, `model`, `thinkingOptionId` and `features`, and no
provider. The handler awaits, in order, `setAgentMode`, `setAgentModel`,
`setAgentThinkingOption` and `setAgentFeature`, then returns
`{success: true}`. There is no rollback, so an exception in a later setter
leaves the earlier ones applied. Notices returned by the setters, such as
"Thinking level applies next turn" (`S/agent/provider-notices.js`), are
discarded.

`S/agent/agent-manager.js:1196` (`setAgentModel`) passes the string to the
provider session and stores it in `config.model` and `runtimeInfo.model`
without checking it against the provider's model list. Reading the model back
therefore returns the request, not a runtime observation.

`S/agent/agent-manager.js:1211` (`setAgentThinkingOption`) does read back:
it takes `getRuntimeInfo().thinkingOptionId` when the provider reports one
and stores it as the effective effort. Codex reports one; Claude does not, so
Claude's effective effort is the requested value.

## Claude sessions

`S/agent/providers/claude/agent.js`:

- `setModel` (`:1910`) first calls `ensureQuery()`, then forwards the model to
  the live SDK query with `setModel`. If an earlier effort change left
  `queryRestartNeeded` set, `ensureQuery()` (`:2479`) retires the
  running process and starts a new one right there, in the middle of the
  current turn. Hence "one call per turn".
- `setThinkingOption` (`:1943`) asserts the effort is supported by the
  *current* model (`assertClaudeThinkingOptionSupported`), sets
  `queryRestartNeeded`, and returns the next-turn notice when a turn is
  active. The restart happens at the next `ensureQuery()`, normally the next
  turn, and the new process is launched with `--effort` and `--model`.
- `setModel` also turns fast mode off when the new model does not support it.

## Codex sessions

`S/agent/providers/codex-app-server-agent.js`:

- `setModel` (`:3415`) stores `config.model`, clears the fast service tier if
  the model does not support it, and recomputes the collaboration mode. The
  model is sent with the next turn.
- `setThinkingOption` (`:3423`) stores the effort and returns the next-turn
  notice during an active turn.
- `getRuntimeInfo` (`:3374`) reports `model` and `thinkingOptionId` from that
  stored config.

## Creating the handoff agent

`create_agent` resolves a required `provider/model` pair. A child on another
provider inherits neither the caller's mode nor its provider options
(`resolveInheritedProviderConfig`, `paseo-tools.js:388`), so it starts in the
entry's `defaultMode` (shown by `paseo provider ls`) unless
`settings.modeId` is given. Settings come only from the call: pass
`settings.thinkingOptionId` explicitly.

## Runtime evidence after the next turn

These read local state only. Run them in the turn *after* the change.

Claude, the model of the latest assistant message:

```sh
f=$(ls "${CLAUDE_CONFIG_DIR:-$HOME/.claude}"/projects/*/"$CLAUDE_CODE_SESSION_ID".jsonl)
jq -r 'select(.type == "assistant") | .message.model' "$f" | tail -1
```

Claude, the flags of the process serving this session (in a Bash tool call
the shell's parent is that process). The full command line contains a Paseo
bearer token inside `--mcp-config`, so print only the flags you need:

```sh
ps -o command= -p "$PPID" | tr ' ' '\n' | grep -x -A1 -E -- '--(model|effort)'
```

A model-only change does not restart the process, so its `--model` stays
stale; use the transcript for the model and the process for the effort.

Codex, the model and effort of the latest turn:

```sh
f=$(find "${CODEX_HOME:-$HOME/.codex}/sessions" -name "*${CODEX_THREAD_ID}.jsonl" | head -1)
jq -c 'select(.type == "turn_context") | .payload | {model, effort}' "$f" | tail -1
```

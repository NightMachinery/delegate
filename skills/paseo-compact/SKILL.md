---
name: paseo-compact
description: Compact the context of a Claude or Codex agent running inside Paseo, your own or another agent's, by sending it the literal prompt "/compact" through Paseo, after persisting critical state to disk. Use it when the user asks you to compact yourself or another Paseo agent; when a long-running agent with a large context reaches a natural endpoint (a task finished, a reply sent while waiting on children or jobs, before an idle stretch or keepalive pings, see paseo-cache-warmer); or when you ask whether a heartbeat can trigger compaction (it cannot, because Paseo wraps heartbeat prompts).
---

# Paseo compact

Paseo passes a prompt that is exactly `/compact` (optionally followed by focus
text) to the agent's CLI as a command, not as a message, so any sender that
can deliver that raw prompt compacts the agent:

```sh
command paseo send <agent-id> --no-wait --prompt "/compact"
```

The MCP tool works the same: `send_agent_prompt` with `prompt: "/compact"`.

## What was verified

All on 2026-10-16 with Paseo daemon 0.10.2:

- **Claude, CLI send, idle agent:** an Opus 5.5 coordinator went from 228,904
  to 26,541 tokens in 80 seconds. Same agent ID, same session; the transcript
  gained a `compact_boundary` record with `trigger: "manual"`.
- **Claude, MCP send, idle agent:** a Haiku 4.5 agent went from 38,279 to 2,356
  tokens in 19 seconds.
- **Claude, focus text:** `/compact Keep only what the story says about the
  blue kettle` reached the summarizer (the summary describes its own request
  as focused on the kettle). Haiku followed it only loosely and still
  summarized the rest, so treat focus text as a hint, not a filter.
- **Claude, self-compaction at a natural endpoint:** the same coordinator,
  later that day, after writing its state and replying to the user: 277,839
  to 25,801 tokens, then it resumed from its state files.
- **Claude, send during the agent's own turn:** see "Compact yourself".
- **Codex:** a gpt-6-luna agent compacted in 3 seconds. Its rollout gained a
  `compacted` record and a `ContextCompaction` item, and it wrote no reply.

## Compact yourself

A long-running agent should compact itself at a **natural endpoint**: a
moment when nothing the next turn needs lives only in its context. Do not
wait for the CLI's automatic compaction. That fires near the context limit,
at a moment the CLI picks, possibly in the middle of a task; and long before
the limit, every request has been paying to read the whole context.

### When

Compact only when the context is at least the **threshold** and more turns
are coming. The threshold is `min_context_tokens` in `~/.compaction.yaml`
(default 128000); create the file with the default when it is missing:

```sh
cfg=~/.compaction.yaml
[ -f "$cfg" ] || printf '%s\n' \
  '# Read by the paseo-compact and paseo-cache-warmer skills.' \
  '# Never compact an agent whose context is below this many tokens,' \
  '# unless the user asks for that compaction explicitly.' \
  'min_context_tokens: 128000' > "$cfg"
min=$(command sed -n 's/^min_context_tokens:[[:space:]]*\([0-9][0-9]*\).*/\1/p' "$cfg" | command head -1)
min=${min:-128000}
```

Below the threshold compaction loses more than it saves:

- The context right after a compaction is not small. The system prompt,
  tools, instruction files and the summary come back, and Claude Code also
  re-attaches the full text of every skill invoked earlier. One coordinator
  measured 25,801 tokens right after compacting and about 100k a few turns
  later.
- Compaction costs one read of the whole context, a summary written as
  output tokens (about five times the input price), and a fresh cache write
  of the new context. Shrinking 100k to 25k pays for itself after about ten
  more requests; shrinking 50k to 25k needs about thirty.
- Every compaction drops detail that the files do not hold.

The threshold applies to compacting other agents too. It is a floor, not a
trigger: above it, still wait for one of these points:

- a task, slice or milestone has just finished, and its result is written
  and committed;
- you have replied to the user and are waiting on something long: child
  agents, CI, a deploy, the user's next message;
- before a long idle stretch, or before keepalive pings start (see
  `paseo-cache-warmer`).

Do not compact in the middle of delicate work whose details live only in
context: a half-done edit, a debugging session whose hypotheses are not
written down, a plan you have not yet saved. Finish it or write it down first.

The size of the context is the sum of `input_tokens`,
`cache_read_input_tokens` and `cache_creation_input_tokens` in the usage of
your latest assistant message (transcript path under "Verify"):

```sh
grep '"type":"assistant"' "$transcript" | tail -1 |
  python3 -c 'import json,sys; u=json.loads(sys.stdin.read())["message"]["usage"]; print(u["input_tokens"] + u["cache_read_input_tokens"] + u["cache_creation_input_tokens"])'
```

### First, persist everything critical to disk

The summary is lossy: treat it as an index, not as memory. Before you start
the waiter, write to files everything the next turn needs:

- the user's instructions that are not yet recorded, verbatim;
- progress and the next step;
- decisions made and their reasons, so they are not reopened;
- the IDs of everything still running: child agents, background jobs,
  heartbeats, schedules, worktrees, and the paths of the result files they
  will write;
- what you learned that is not obvious: causes of errors, traps, measured
  numbers;
- uncommitted work: commit it, or note where it is and what state it is in.

Then ask: could a fresh agent continue from these files alone? If not, the
compaction will lose something. A short resume recipe in the project's
`CLAUDE.md` or `AGENTS.md` (which files to read, what to check before acting)
survives intact, because those files are loaded again after compaction.

### Then let the turn end before the compaction starts

A Paseo prompt sent to an agent that is still running **interrupts its turn**.
Tested: an agent ran `paseo send "$PASEO_AGENT_ID" --prompt "/compact"` in
the middle of its turn. Paseo cut the turn off at once. The transcript shows
the tool call that ran the send as rejected and "[Request interrupted by user
for tool use]", the two commands the agent meant to run next never ran, and
the compaction followed as a new turn.

So an agent can compact itself mid-turn, but only by killing the rest of
that turn, including whatever it was about to write or report. The clean way
is to let the turn finish first: start a background job that outlives the
turn (in Claude Code, a Bash call with `run_in_background`), end the turn,
and let the job send `/compact` once Paseo shows the agent idle. The
compaction then runs as a turn of its own, with nothing to interrupt. That
is how the 228k-token coordinator above was compacted: it started the job,
replied to the user and went idle; ten seconds later the job sent `/compact`.

1. Read your ID from `PASEO_AGENT_ID`; confirm it is non-empty and is you
   (the `paseo-model-update` skill has the identity check). Never derive it
   from a title.
2. Start the waiter, then end the turn with a short reply that tells the
   user you are compacting:

```sh
id=$PASEO_AGENT_ID
status() { command paseo inspect "$id" --json 2>/dev/null |
  python3 -c 'import json,sys; print(json.load(sys.stdin).get("Status"))'; }
n=0
until [ "$(status)" = idle ] || [ "$n" -ge 180 ]; do n=$((n + 1)); sleep 10; done
sleep 20
[ "$(status)" = idle ] && command paseo send "$id" --no-wait --prompt "/compact"
```

### After the compaction

The job's completion notice reaches you after the compaction, as a
background-task event. It is your first wake-up with the new context:

1. Check that the compaction happened (see "Verify").
2. Read your state files again before acting, and follow the resume recipe
   if the project has one.
3. Check what is actually running (child agents, background jobs) before
   you rely on the summary: anything it calls running may have finished, and
   nothing still running should be started a second time.

## Compact another agent

1. `paseo inspect <id> --json` gives `Provider` and `Status`. Send only when
   the status is `idle`; a send to a `running` agent interrupts it.
2. If the agent holds state only in its context, tell it first to write that
   state to its files.
3. Send `/compact`, then verify.

A finish notification after a Claude compaction repeats the agent's previous
reply, because compaction adds no assistant message; do not read it as a new
answer. No finish notification arrived for the Codex compaction.

## What cannot deliver it

- **Heartbeats.** Paseo wraps a heartbeat prompt in its own text (`Schedule
  "<name>" fired (id=..., run=...)`, then the prompt), so `/compact` is not
  the whole message and arrives as plain text.
- **Schedules.** `create_schedule` starts a fresh agent on each run, which
  has nothing to compact.

## Verify

The session ID is `persistence.sessionId` from `get_agent_status`.

- **Claude:** the transcript is
  `<config dir>/projects/<cwd with each / replaced by ->/<session id>.jsonl`,
  where the config dir is the provider's `CLAUDE_CONFIG_DIR` (default
  `~/.claude`). The last compaction record shows trigger and token counts:

```sh
grep '"subtype":"compact_boundary"' "$transcript" | tail -1 |
  python3 -c 'import json,sys; m=json.load(sys.stdin)["compactMetadata"]; print(m["trigger"], m["preTokens"], m["postTokens"])'
```

- **Codex:** the rollout is
  `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*-<session id>.jsonl` (default
  `~/.codex`); look for a line with `"type":"compacted"`.

## When it pays

On a subscription the main conversation's prompt cache lives one hour. A
cache read costs about a tenth of the input price and a one-hour cache write
about twice it, so the size of the context sets the price of every later
request and of every cold restart. Compacting 229k tokens to 27k made both
about eight times cheaper.

- Compact right after a turn, while the cache is still warm. Compaction reads
  the whole context once; after an idle hour that read is itself a cold
  rebuild of the large context.
- To keep the cache warm across idle stretches, use the `paseo-cache-warmer`
  skill, which compacts with this one first.
- Claude Code's native idle compaction (`idleCompaction` in settings.json) is
  documented for terminal sessions. Do not count on it for an agent that
  Paseo runs through the SDK.

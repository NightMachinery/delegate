---
name: paseo-compact
description: Compact the context of a Claude or Codex agent running inside Paseo, your own or another agent's, by sending it the literal prompt "/compact" through Paseo. Use it when the user asks you to compact yourself or another Paseo agent, when a long-running agent should shrink its context before an idle stretch or before keepalive pings (see paseo-cache-warmer), or when you ask whether a heartbeat can trigger compaction (it cannot, because Paseo wraps heartbeat prompts).
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
- **Claude, send during the agent's own turn:** see the next section.
- **Codex:** a gpt-6-luna agent compacted in 3 seconds. Its rollout gained a
  `compacted` record and a `ContextCompaction` item, and it wrote no reply.

## Why self-compaction goes through a background job

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

1. Write durable state first: progress, open tasks, the IDs of running jobs
   and agents, anything the next turn needs. The summary is lossy; treat it
   as an index, not as memory.
2. Read your ID from `PASEO_AGENT_ID`; confirm it is non-empty and is you
   (the `paseo-model-update` skill has the identity check). Never derive it
   from a title.
3. Start the waiter, then end the turn with a short reply:

```sh
id=$PASEO_AGENT_ID
status() { command paseo inspect "$id" --json 2>/dev/null |
  python3 -c 'import json,sys; print(json.load(sys.stdin).get("Status"))'; }
n=0
until [ "$(status)" = idle ] || [ "$n" -ge 180 ]; do n=$((n + 1)); sleep 10; done
sleep 20
[ "$(status)" = idle ] && command paseo send "$id" --no-wait --prompt "/compact"
```

The job's completion notice reaches you after the compaction, as a
background-task event.

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
- Compact before a long idle stretch, or before keepalive pings start, not
  in the middle of delicate work whose details live only in context.
- To keep the cache warm across idle stretches, use the `paseo-cache-warmer`
  skill, which compacts with this one first.
- Claude Code's native idle compaction (`idleCompaction` in settings.json) is
  documented for terminal sessions. Do not count on it for an agent that
  Paseo runs through the SDK.

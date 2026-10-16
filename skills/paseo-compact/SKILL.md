---
name: paseo-compact
description: Compact the context of a Claude agent running inside Paseo, your own or another agent's, by sending it the literal prompt "/compact" through Paseo while it is idle. Use it when the user asks you to compact yourself or another Paseo agent, when a long-running coordinator should shrink its context before an idle stretch so cache keepalives and cold restarts cost less, or when you ask whether a heartbeat can trigger compaction (it cannot, because Paseo wraps heartbeat prompts).
---

# Paseo compact

A Claude agent under Paseo runs Claude Code through the Agent SDK, and the SDK
treats a prompt that is exactly `/compact` as the compaction command. So any
sender that can deliver that raw prompt to an **idle** agent can compact it:

```sh
command paseo send <agent-id> --no-wait --prompt "/compact"
```

Verified on 2026-10-16 with Paseo daemon 0.10.2 and a Claude Opus 5.5 agent:
the agent kept its ID and session, its transcript gained a
`compact_boundary` record with `trigger: "manual"` (228,904 tokens before,
26,541 after, 80 seconds), and its next turn began from the summary, with a
`<command-name>/compact</command-name>` note and the output `Compacted`.

## What cannot deliver it

- **Heartbeats.** Paseo wraps a heartbeat prompt in its own text (`Schedule
  "<name>" fired (id=..., run=...)`, then the prompt), so `/compact` is not
  the whole message and arrives as plain text.
- **Schedules.** `create_schedule` starts a fresh agent on each run, which
  has nothing to compact.
- **A send while the agent is running.** Not tested. It may interrupt the
  running turn or be queued; wait for `idle` instead.

## Compact another agent

1. Check it is a Claude-provider agent and idle: `paseo inspect <id> --json`
   gives `Provider` and `Status` (`running` or `idle`).
2. Tell it first, if it holds state only in its context, to write that state
   to its files. The summary is lossy.
3. Send `/compact` as above, and verify (below).

## Compact yourself

You cannot compact during your own turn: the prompt must arrive after the
turn ends. Hand the send to a background job that outlives the turn (in
Claude Code, a Bash call with `run_in_background`), then end the turn.

1. Write durable state first: progress, open tasks, the IDs of running jobs
   and agents, and anything else the next turn needs. Treat the summary as a
   loose index, not as memory.
2. Read your ID from `PASEO_AGENT_ID` and confirm it is non-empty and is you
   (the `paseo-model-update` skill has the identity check). Never derive it
   from a title.
3. Start the waiter, then finish the turn with a short reply:

```sh
id=$PASEO_AGENT_ID
status() { command paseo inspect "$id" --json 2>/dev/null |
  python3 -c 'import json,sys; print(json.load(sys.stdin).get("Status"))'; }
n=0
until [ "$(status)" = idle ] || [ "$n" -ge 180 ]; do n=$((n + 1)); sleep 10; done
sleep 20
[ "$(status)" = idle ] && command paseo send "$id" --no-wait --prompt "/compact"
```

The job's completion notice reaches you after the compaction, as an
ordinary background-task event.

## Verify

The Claude transcript is
`~/.claude/projects/<cwd with each / replaced by ->/<session id>.jsonl`; the
session ID is `persistence.sessionId` from `get_agent_status`. The last
compaction record shows the trigger and the token counts:

```sh
grep '"subtype":"compact_boundary"' "$transcript" | tail -1 |
  python3 -c 'import json,sys; m=json.load(sys.stdin)["compactMetadata"]; print(m["trigger"], m["preTokens"], m["postTokens"])'
```

## When it pays

On a subscription the main conversation's prompt cache lives an hour, and
every request that reads it restarts the hour. A cache read costs about a
tenth of the input price and a one-hour cache write about twice it, so the
size of the context sets the price of every keepalive ping and of every cold
restart. Compacting a 229k-token context to 27k made both roughly eight times
cheaper.

- Compact while the cache is still warm, right after a turn. Compaction reads
  the whole context once; after an hour of idleness that read is itself a
  cold rebuild of the large context.
- Compact before a long idle stretch or before keepalive pings start, not in
  the middle of delicate work whose details live only in context.
- Claude Code's native idle compaction (`idleCompaction` in settings.json) is
  documented for terminal sessions. Do not count on it for an agent Paseo runs
  through the SDK.

## Not yet tested

- Codex agents. Codex has its own `/compact` in its TUI; whether Paseo's
  Codex path honors it is unknown. Try it on a throwaway agent first.
- `/compact <focus instructions>` through Paseo.
- The MCP `send_agent_prompt` tool instead of the CLI. It likely behaves the
  same; the CLI call above is the verified one.

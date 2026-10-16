---
name: paseo-cache-warmer
description: Keep a long-lived Claude agent's prompt cache warm inside Paseo with keepalive heartbeats, and decide when that pays. Use it when a coordinator or other long-running Paseo agent will sit idle between wake-ups (children finishing, background jobs, user messages) and each cold restart would rebuild a large context, when the user asks about keepalive pings or cache warming, or when you must decide between warming the cache, compacting first, and letting it go cold. It reuses the paseo-compact skill to shrink the context before warming.
---

# Paseo cache warmer

A Claude agent's conversation is cached between requests. When the cache is
warm, a wake-up reads the whole context at a tenth of the input price; when
it has expired, the wake-up writes the context into the cache again at about
twice the input price. For a coordinator with a few hundred thousand tokens
of context, one cold wake-up costs as much as twenty warm ones.

A **keepalive** is a heartbeat that makes the agent do one cheap request
before the cache expires. This skill sets keepalives up, tears them down, and
says when they pay. To make every keepalive cheaper, it shrinks the context
first with the `paseo-compact` skill.

## Facts this rests on

- On a Claude subscription the main conversation's cache lives one hour,
  and every request that reads it starts the hour again. On paid overage it
  drops to five minutes, and keepalives stop paying.
- A cache read costs about 0.1 times the input price; a one-hour cache write
  about 2 times. So twenty reads cost about one rebuild.
- How a subscription plan counts cache reads against its usage limits is not
  published. Watch the usage meter while keepalives run.
- Paseo heartbeats take cron expressions only, are scoped to the agent that
  created them, and do not appear in `list_schedules`. Create them with
  `create_heartbeat`, remove them with `delete_heartbeat`; there is no update,
  so change one by deleting and recreating it.
- Paseo wraps each heartbeat prompt in its own text (`Schedule "<name>"
  fired ...`), so a heartbeat cannot carry a slash command such as
  `/compact`.
- Codex agents are not covered: their prompt caching is the provider's own
  and has no lifetime you can extend this way.

## Decide first

1. **Will the agent certainly be woken within about 15 hours?** Keepalives
   every 45 minutes cost about one cold rebuild per 15 hours. If nothing is
   running that will wake it (no children, no background jobs, no user
   expected), do not warm: let the cache go cold, and compact first so the
   eventual cold start is small.
2. **Is the context large?** If it is at least the threshold in
   `~/.compaction.yaml` (`min_context_tokens`, default 100k; `paseo-compact`
   says how to read and create it) and holds little that the next turns need
   verbatim, compact it before warming: follow `paseo-compact` ("Compact
   yourself"). Compaction divides the cost
   of every later keepalive and of any cold start by the same ratio (229k to
   27k tokens in one test, about eight times). Compact right after a turn,
   while the cache is still warm; compaction reads the whole context once,
   and after an idle hour that read is itself a cold rebuild.
3. **Is delicate work in flight whose details live only in context?** Then
   warm without compacting until it lands, and write its state to files.

## Set keepalives up

1. Write the recipe to a file first (cron lines, timezone, the exact prompt,
   the current heartbeat IDs), so recreating them later is exact. Update the
   IDs every time you recreate them.
2. Create three heartbeats. Cron cannot say "every 45 minutes", but three
   heartbeats on a three-hour cycle can, at exact 45-minute gaps around the
   clock:

```text
keepalive-a   7,52 0-23/3 * * *
keepalive-b   37 1-23/3 * * *
keepalive-c   22 2-23/3 * * *
```

   45 minutes leaves a 15-minute margin for a ping that waits behind a busy
   daemon. Pass your `timezone` so the cron hours mean what you think.
3. Give all three the same prompt. Keep it short and make the reply shorter,
   because everything a ping adds to the context is paid for on every later
   request. A template:

> Keepalive ping for <project>: three heartbeats named <name>-a/b/c fire
> every 45 minutes to keep this agent's prompt cache warm. Use ONE shell
> call that would reveal trouble (for example the date, the run state's
> mode line, git status, and whether expected workers are alive). Act only
> on a discrepancy: a worker the state file calls active has finished (fold
> its result in and continue), or stray processes nobody owns. If nothing is
> running at all, delete the three heartbeats (IDs in <recipe file>), say
> so, and recreate them from that file when work starts again. Otherwise
> reply only "ping ok", with no further tool calls.

## Tear them down and bring them back

- Keepalives exist only while work runs. The ping itself deletes all three
  when it finds nothing running; do the same yourself when the last worker
  finishes and no reply is expected.
- When work starts again (you launch a child, start a long background job,
  or the user is active), recreate the three from the recipe file before
  the next idle stretch, and record the new IDs.
- Before a long idle stretch with no keepalives, consider compacting
  (decision step 1).

## Check that it works

Each ping should read almost everything from the cache and write very
little. The usage of the agent's latest assistant message in its transcript
shows it (the transcript path is in `paseo-compact`, "Verify"):

```sh
grep '"type":"assistant"' "$transcript" | tail -1 |
  python3 -c 'import json,sys; u=json.loads(sys.stdin.read())["message"]["usage"]; print(u["cache_read_input_tokens"], u["cache_creation_input_tokens"], u.get("cache_creation"))'
```

A warm ping shows a large read and a small write. A large write
(`ephemeral_1h_input_tokens` close to the context size) means the cache had
already expired: the gap was too long, or the account fell back to the
five-minute lifetime. Check this on the first pings of every new setup.

## Not yet tested

- What a heartbeat does when it fires while the agent is mid-turn. A Paseo
  prompt sent to a running agent interrupts its turn (see `paseo-compact`);
  whether heartbeats wait or interrupt is unknown. Keep pings rare enough
  that a collision is unlikely, and if one interrupts a turn, say so in the
  run state.
- A plugin that warms each opted-in agent from Paseo's turn events, without
  heartbeats. None exists yet.

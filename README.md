# delegate

Agent skills for choosing who does the work: which model, which account, which
execution backend, and, inside Paseo, which model and effort the current agent
itself runs. They are policy skills: they decide and brief, and leave execution
to the backend's own skill.

- `delegate` selects a weaker, peer, or stronger worker, keeps account choice
  separate from capability, and picks an in-process, Paseo, or tmux backend.
  Model routing is in `skills/delegate/references/models.md`, the Paseo binding
  in `references/paseo.md`.
- `delegate-weaker` and `weaker` are compatibility entrypoints that select
  weaker mode. They link to `../delegate/` by relative path, so install them
  together with `delegate`.
- `paseo-model-update` answers "switch yourself to <model> <effort>" and "what
  model are you running?" inside Paseo. It checks read-only first, changes model
  and effort in place on the agent's own provider entry, and otherwise offers a
  handoff, since Paseo cannot move a running agent to another provider. The
  Paseo 0.10.2 behavior it relies on is cited in its
  [internals reference](skills/paseo-model-update/references/paseo-internals.md).

The full guide is [docs/delegate.md](docs/delegate.md).

## Install

```sh
npx skills add NightMachinery/delegate --global \
  --agent claude-code codex --skill delegate delegate-weaker weaker paseo-model-update
```

For another Claude profile, rerun with `CLAUDE_CONFIG_DIR` set to its config
directory and `--agent claude-code`, one profile per invocation.

For local development, symlink each directory under `skills/` into
`~/.agents/skills/<name>` and each Claude profile's `skills/<name>`, without
overwriting an existing installation. On the author's machine
`agent-skills-link` does this for every configured agent, globbing
`~/code/skills/*/skills/*/SKILL.md`. Skill names must be unique across those
repositories, so never keep a second copy of these directories elsewhere.

## Backends

These skills name their backends and do not bundle them:

- **tmux:** the native interactive backend is the `tmux-subagents` skill from
  [NightMachinery/tmux-subagents](https://github.com/NightMachinery/tmux-subagents).
  It works standalone and reads `delegate` when installed.
- **Paseo:** use the official `paseo` skill that Paseo installs. The Paseo
  binding here adds account and ownership rules; it does not mirror Paseo's
  command reference.
- **In-process:** the runtime's own agent tool.

Research-run companions, such as durable run state and quota-aware
coordinator handoff, are in
[NightMachinery/research-agent-ops](https://github.com/NightMachinery/research-agent-ops).

## History

Extracted on 2026-10-03 from NightMachinery/tmux-subagents at commit
`3ae9df6`, with the history of these paths kept by `git filter-repo` on a fresh
clone. The tmux-subagents history itself was not rewritten.

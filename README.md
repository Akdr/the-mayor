# the-mayor

A Claude Code plugin for [Orca](https://github.com/stablyai/orca) users. One Claude Code session, **The Mayor**, runs in a hub checkout. It spawns every other agent into its own Orca worktree and terminal, watches them, relays decisions, and cleans up after a merge. The work stays visible: each agent is a pane you can open and type in, not a hidden subagent.

This is opinionated. It encodes one person's workflow. Take what fits.

## What it does

- Spawns one Claude Code agent per repo in an Orca worktree, with a full task brief in the first message.
- Waits for agents in the background and wakes when a turn ends or an agent is blocked on a question, so the hub does not go silent.
- Reads Orca's agent state (`working`, `permission`, `active`) instead of guessing from the screen.
- Verifies "done" claims against the artefact before it acts on them.
- Brings every decision to you as a numbered list.
- Captures findings from an agent before it closes the pane, then removes the worktree safely.

## Requirements

- [Orca](https://github.com/stablyai/orca) with the `orca` CLI on `PATH`. Tested with the `orca` CLI 1.4.224.
- `jq` and, for the PR steps, the GitHub CLI `gh`.
- If you run Claude Code in a sandbox, allow reading `~/.orca`. Orca's Claude Code hooks run scripts from `~/.orca/agent-hooks/`. Without them, Orca reports no agent state and the skill falls back to screen polling.

## Install

```
/plugin install the-mayor --marketplace Akdr/the-mayor
```

On older Claude Code versions:

```
/plugin marketplace add Akdr/the-mayor
/plugin install the-mayor@the-mayor
```

## Use

Start Claude Code in an Orca terminal in your hub checkout, then ask for what you want: "spawn an agent for PROJ-123 in api-service", "status?", "close out the merged PRs". The skill triggers on phrases like these.

Project rules go in a project skill with the same name, `.claude/skills/the-mayor/SKILL.md`, or in `CLAUDE.md`. They add to the plugin's rules, for example branch patterns, secrets or release steps.

## License

MIT

# Orca command reference for The Mayor

Checked against the `orca` CLI 1.4.224 (`orca skills get orca-cli` and `--help`). When a flag here
is rejected, report `$ORCA_APP_VERSION` and `orca --version`, then use `--help`. Do not
guess flags. `orca skills get orca-cli` prints the guide for the installed binary.

## Addresses

- **Worktree id** is `<repoId>::<worktreePath>`. Copy the whole `id` from JSON. A bare repo
  id names only the repo.
- Selectors: `id:<id>`, `path:<absolute path>`, `name:<displayName>`, `branch:<branch>`,
  `active`. Prefer `path:` or `id:`; display names repeat.
- **Terminal handle** `term_...` addresses one terminal. It is runtime-scoped.
- Fixed shapes: the hub is the checkout the user opened; a spawned agent lives under Orca's
  workspace root, by default `~/orca/workspaces/<repo>/<name>`.

Your own context:

```bash
printf '%s\n' "${ORCA_WORKTREE_ID:-}" "${ORCA_TERMINAL_HANDLE:-}"
orca worktree current --json | jq -r '.result.worktree | "\(.id) \(.displayName)"'
```

## Find

```bash
orca repo list --json | jq -r '.result.repos[] | "\(.id) \(.path) \(.gitUsername // "")"'
orca worktree ps --json | jq -r '.result.worktrees[] | "\(.status)\t\(.liveTerminalCount)\t\(.path)"'
orca worktree ps --json | jq -r '.result.worktrees[] | select(.path=="<hub path>") | .childWorktreeIds[]'
orca terminal list --worktree path:<worktree path> --json \
  | jq -r '.result.terminals[] | "\(.handle) \(.connected)"'
```

`status` is `working`, `permission`, `active` or `inactive`. `permission` means an agent
waits on a question, a menu or a permission prompt. `.agents[]` holds one entry per Claude
pane, with `state` (`working`, `waiting`, `done`), `toolName`, `prompt` and
`lastAssistantMessage`. It comes from Claude Code hooks that Orca installs; they run
scripts under `~/.orca/agent-hooks/`, so a sandbox must allow reading `~/.orca`. An empty
array means the hooks are blocked.

| `.agents[].state` | `worktree.status` | Meaning |
|---|---|---|
| `working` | `working` | A turn runs. |
| `waiting` | `permission` | Blocked on a question or a prompt. `toolName` names it, e.g. `AskUserQuestion`. |
| `done` | `active` | The turn ended. `lastAssistantMessage` holds the final reply. |

A new agent reads `done` before its first prompt starts, so `done` proves a finished run only
after the state was `working` once.

```bash
orca worktree ps --json | jq -r --arg p "<worktree path>" '.result.worktrees[]
  | select(.path==$p) | "\(.status) \([.agents[].state] | join(","))"'
```

## Spawn

```bash
orca repo add --path <repo path> --json          # once per repo
out=$(orca worktree create --repo path:<repo path> \
  --name <KEY>-<slug> --agent claude --parent-worktree active --json)
echo "$out" | jq -r '"\(.result.worktree.id) \(.result.worktree.branch) \(.result.agentTerminalHandle // .result.startupTerminal.handle // "NO-HANDLE") \(.error.message // "")"'
```

- `--agent claude` launches the agent in the first terminal. `--prompt "<text>"` sends an
  initial brief to it. `--parent-worktree active` records the hub as parent, so the hub's
  `childWorktreeIds` lists it.
- `--agent` has no model flag. Orca starts the command set in Settings
  (the agent command override) with the default args set there. Pinning a model there is the
  user's call.
- Orca prefixes the branch with the repo's `gitUsername` (`<user>/PROJ-123-add-header`). A
  repo with an empty `gitUsername` gets branches that match `--name`. The worktree folder
  keeps the bare name. Read `branch` from the result.
- Omit `--base-branch` unless the user asks for stacked work. Do not pass `--activate`.
- A bare create (no `--agent`) opens a fallback shell. Close that shell only after
  `terminal list` shows it is unused.

Respawn on the right model when the banner is wrong:

```bash
orca terminal close --worktree id:<id> --all --json
orca worktree rm --worktree id:<id> --json
orca worktree create --repo path:<repo path> --name <KEY>-<slug> --parent-worktree active --json
orca terminal create --worktree id:<new-id> --command "claude --model <model>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
orca terminal read --terminal <handle> --screen --json          # status line names <model>
```

A second agent in an existing checkout is a terminal, not a worktree:

```bash
orca terminal create --worktree path:<worktree path> --command claude --json
```

## Wait, send, read

```bash
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
orca terminal send --terminal <handle> --text "<text>" --enter --wait-submit 10 --json
orca terminal send --terminal <handle> --enter --json                 # bare Enter
orca terminal send --terminal <handle> --text "2" --json              # AskUserQuestion choice: no --enter
orca terminal send --terminal <handle> --text "<text>" --enter --retry-request <id> --json
orca terminal read --terminal <handle> --screen --json
orca terminal read --terminal <handle> --json
orca terminal read --terminal <handle> --cursor <cursor> --limit 1000 --json
```

- `wait` prints a normal result on timeout. Read `.result.wait.satisfied`.
  - **Startup wait** (120000 ms, before the brief): on `false`, rerun once with a larger
    `--timeout-ms`; if still false, report the agent as not started.
  - **Run wait** (900000 to 1800000 ms): on `false`, read the screen and arm the wait again.
    A busy agent is not a dead one. Run it with the Bash tool's `run_in_background: true`: a
    foreground call stops at 600000 ms, and only a background command that exits wakes The
    Mayor after its turn ends.
- `tui-idle` fires while a new agent runs its SessionStart hooks, before a
  `worktree create --prompt` brief starts. After such a create, the first `tui-idle` is not
  the end of the run. Wait on the Orca state instead (the-mayor SKILL, "Wait in the
  background").
- `send`: `.result.send.accepted` proves input acceptance. `turn_started` in
  `.result.send.prompt.stages` proves the turn. A default send observes for 0 seconds, so a
  receipt that stops at `input_accepted` is "unproven", not "failed". `--wait-submit <s>`
  observes the same prompt; it never resends. Never resend on silence.
- `read --screen` returns the rendered frame; `--screen` and `--cursor` are exclusive. The
  stream form returns accumulated output, and a TUI repaint comes back as stacked fragments.
  Use `--screen` for a question, a menu, the model banner, and before every send.
- The stream tail is in `.result.terminal.tail`; page from `oldestCursor`, then follow
  `nextCursor` while `limited` is true and `nextCursor` differs from `latestCursor`.

Useful one-liners:

```bash
# the rendered screen as text (a --screen read puts the frame in .tail, source: screen)
orca terminal read --terminal <h> --screen --json \
  | jq -r '.result.terminal.tail | if type=="array" then join("\n") else . end'
# the prompt box: empty "❯" means idle and nothing left unsent
orca terminal read --terminal <h> --screen --json \
  | jq -r '.result.terminal.tail | if type=="array" then join("\n") else . end' | grep -E '^❯' | tail -1
# the stream tail, trimmed of status noise
orca terminal read --terminal <h> --json \
  | jq -r '.result.terminal.tail | if type=="array" then join("\n") else . end' \
  | grep -v '^\s*$' | grep -vE '^\s*─+$|^\s*\[(Opus|Sonnet|Haiku)|bypass permissions' | tail -40
```

## Dismantle

```bash
orca terminal close --worktree id:<repoId>::<path> --all --json
orca worktree rm --worktree id:<repoId>::<path> --json
orca worktree rm --worktree id:<repoId>::<path> --force --json      # only on a checked tree
```

- `close --worktree --all` stops every terminal process in that workspace and removes its
  tabs, layouts and resume records. When Orca cannot confirm every PTY stopped, the result is
  `unverifiable`; do not report the processes as exited.
- `rm` removes the worktree from Orca and git, and deletes the local branch only when Orca
  can prove it merged. `--force` forces the worktree removal, not the branch deletion.
- Use workspace Sleep (the user, in the app) when the agent should resume later. `terminal
  stop` is legacy; do not use it.

## Card state (optional, cheap)

```bash
orca worktree set --worktree id:<id> --comment "PR #177 open, waiting for review" --json
orca worktree set --worktree id:<id> --workspace-status in-review --json
orca worktree set --worktree id:<id> --pr 177 --json
```

The comment is the status line on the workspace card. A failed comment update is not an
error to surface.

## Shell gotchas (zsh, and scripts under bash `set -euo pipefail`)

- Never name a variable `path`; it shadows `PATH` in zsh and every later command fails.
  `end` is a reserved word.
- Iterate pairs as lines: `printf '%s\n' "a 1" "b 2" | while read -r name id; do ...; done`.
  Never `set -- $pair`; zsh does not word-split an unquoted expansion. Use `${=var}` when
  you need it.
- `$?` after a pipeline is the last command's status. Capture first:
  `out=$(cmd 2>&1); code=$?`.
- `until CMD; do sleep; done` exits when CMD is true. Write the state you wait for:
  `until [ "$(gh pr view N --json state --jq .state)" = "MERGED" ]`.
- Under `set -e`, a trailing `[[ cond ]] && x=y` that is false aborts the script. Write a
  full `if`. End a `for` loop whose last command is a test with `:`.
- Under `pipefail`, `git show <ref>:<path>` exits 128 when the path is absent. Wrap it:
  `{ git show ... || true; } | ...`.
- A long brief goes through a heredoc into a variable, then `--text "$T"`. Quote it once.

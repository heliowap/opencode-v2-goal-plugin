> **Moved.** This plugin now lives in [heliowap/opencode-plugins](https://github.com/heliowap/opencode-plugins/tree/main/plugins/goal). Install: `opencode plugin add 'github:heliowap/opencode-plugins#main::path:plugins/goal'`. This repository is archived.

# opencode-v2-goal-plugin

> [!NOTE]
> This is a community plugin. It is not built by the OpenCode team and is not affiliated with OpenCode in any way.

Codex-style `/goal` for OpenCode 2. A persistent, per-session objective that keeps the agent working across turns until completion is audited against evidence.

## Install

Requires OpenCode 2.

```sh
opencode plugin add github:heliowap/opencode-v2-goal-plugin
```

This installs the plugin and adds it to your global `opencode.jsonc`. If `/goal` does not show up, run `opencode service restart`. Update with `opencode plugin update` and uninstall with `opencode plugin remove github:heliowap/opencode-v2-goal-plugin`.

## Compatibility

| Surface | Status |
|---|---|
| OpenCode 2.0.24 | Verified by 39 host canaries against a real `opencode serve`, graded on stored state. They run in CI on every push. |
| Status above the input, objective in the sidebar, `/goal edit` editor, and terminal UI controls for token budget, pause, and clear | Verified by 8 canaries on OpenCode 2.0.24, the version CI pins, that drive the real `opencode` terminal in tmux, with keys and mouse clicks, and read the screen. The reducer and RPC tests cover the other controls. |
| Headless `opencode run` | Works against a running server (`--server` or the background service). With `--standalone` the goal stops after the first turn, see [Headless runs](#headless-runs) |
| Live models | `devin/swe-2`, `gemini-3.8-flash`, and the free `nemotron-3.5-lightning-free` complete real goals, stop at budgets, and survive a restart |
| `/goal` typed in the terminal UI | Only `/goal edit` is typed in the real terminal by a canary; the goal controls cover the other changes without typing |
| Desktop and web apps, Windows, other OpenCode versions | Not established |

Details, session evidence, and what is not proven: [`docs/compatibility.md`](docs/compatibility.md).

## Use

```text
/goal make the checkout suite pass without changing the public API
/goal --tokens 250K migrate the tests from Jest to Vitest
/goal --verify "bun test" make the checkout suite pass
/goal                    show status and token usage
/goal edit               open the current objective in an editor (terminal UI)
/goal edit <objective>   change the objective, keeping usage
/goal pause
/goal resume
/goal clear
```

In the terminal UI, a line above the input shows the goal of the open session:

| Goal | Line |
|---|---|
| Active, turn running | `◐ goal running · 12K / 50K` |
| Active, between turns | `◎ goal active · 12K / 50K` |
| Paused | `⏸ goal paused` |
| Blocked | `⚠ goal blocked` |
| Out of budget | `goal budget reached · 52K / 50K` |
| Complete, or no goal | nothing |

While a turn runs, the first glyph turns through `◐ ◓ ◑ ◒`, about every 150 ms.

The session sidebar shows the objective of an unfinished goal at its bottom, next to the input, cut to three lines. Its `Goal` title spins like the line above the input while a turn runs.

`/goal edit` without an objective, or a click on the sidebar's `Goal` title, opens the full objective in an editor dialog. `ctrl+s` saves it, `esc` discards the change. Clients without the terminal UI, such as `opencode run`, need `/goal edit <objective>`.

Clicking the line above the input, or running `Goal controls` from the command palette (`ctrl+p`), opens the goal controls: pause or resume, edit the objective, set or remove the token budget (`50K`, `1.5M`, empty for no limit) and the verify command, and clear the goal after a confirm. Raising the budget of a goal that ran out resumes it. Until then, an out-of-budget goal offers only the budget, verify, and clear controls. Creating a goal still takes `/goal <objective>`.

The agent can also create, read, and finish goals with the `goal_create`, `goal_get`, and `goal_update` tools. `goal_get` also returns the `/goal` syntax, so the agent can answer questions about the command.

## How it works

- State lives in the plugin's `ctx.storage` under `goal/<sessionID>` and survives restarts.
- When a turn ends with the goal `active`, the plugin waits quietly for 120 seconds before sending the synthetic continuation that starts the next turn. The `continuationDelayMs` plugin option accepts a finite, nonnegative delay in milliseconds (host canaries use `0`). Explicit set/resume and bounded server-error retries do not wait for this cooldown. A new turn invalidates the pending continuation; a location reload preserves its deadline. The message carries the full instructions: keep the scope intact, work from current evidence, check for progress, and audit completion requirement by requirement before calling `goal_update`.
- Three consecutive turns with no visible assistant text and no successful `write`, `edit`, `patch`, or `apply_patch` tool call mark the goal `blocked`, so observation-only polling cannot spin. Shell calls and unknown tools conservatively do not count as progress; visible text still counts, so the cooldown also limits narrated polling.
- The model may mark the goal `blocked` only after the same blocker repeats for three turns, and `paused` only at the user's request.
- Token usage is counted from each model step (input, output, and reasoning tokens). Crossing `--tokens` marks the goal `budget_limited` and steers the agent to wrap up.
- With `--verify "cmd"`, `goal_update` with `complete` first runs `cmd` with `sh -c` in the goal's directory. The goal completes only if it exits 0. Otherwise the goal stays open, the plugin counts the failed check, and the model gets the exit code and the end of the output. Interrupting the turn kills the command. Only the user can set the command: `goal_create` cannot, because a plugin tool cannot ask for `bash` permission per call.
- A user interrupt pauses the goal. A turn that fails with an HTTP 5xx error, after OpenCode's own retries, is retried up to three times, 30 s, 2 min, and 5 min later; the goal stays active meanwhile. Any other failure, including a rate limit (429), and a fourth server error in a row block the goal so retries cannot loop.
- Turns run by the `plan` agent neither count nor continue the goal.
- A subagent that shows no activity for 10 minutes is interrupted, and its parent gets a queued `[goal] Watchdog:` notice. The `watchdogStallMs` plugin option sets that threshold in milliseconds. At the default threshold the watchdog checks every 60 s, so detection lands between 10 and 11 minutes after the last activity. It never interrupts a child with an open tool call, however long the call runs. If the interrupt itself fails, the notice says the child could not be interrupted and may still be running, so the parent checks it before redispatching.
- The loop guard watches shell commands. The third identical command in a row (whitespace at the ends ignored) gets a loop warning appended to its result, telling the agent to change course instead of polling. It warns once per streak and never blocks a call. A file edit (`write`, `edit`, `patch`, `apply_patch`) or a subagent dispatch starts a new streak; reads and searches do not.

All state rules live in the pure reducer `step` in `src/goal.ts`. `src/index.ts` only translates OpenCode events into `GoalEvent`s and runs the returned `GoalEffect`.

## Trace log

The plugin writes one JSON line per goal event to `$XDG_DATA_HOME/opencode/log/goal.jsonl` (`~/.local/share/opencode/log/goal.jsonl` by default), next to OpenCode's own log:

- **Status changes and continued turns:** the event, the statuses before and after, the effect, token usage, and the note. A new or edited goal also records its objective.
- **Every typed `/goal`:** the text as typed and what it parsed to, including refused commands.
- **Every failed `goal_create`, `goal_get` or `goal_update` call,** with its error.
- **Plugin failures** that the plugin caught, with their session when there is one.

At 100 MB the file moves to `goal.1.jsonl`, replacing the previous one, so the log never takes more than 200 MB and the oldest records are deleted first. Several OpenCode processes on one machine share the file. If two of them rotate at the same moment, the older half can be deleted early.

Summarize it with `bun run report` in a clone of this repository, or `bun run report <sessionID>` for one session's timeline.

Either of these turns it off: the plugin option `"log": false`, or `OPENCODE_GOAL_LOG=0` (or `false`) in the environment of the OpenCode server.

```jsonc
{ "plugins": [{ "package": "github:heliowap/opencode-v2-goal-plugin", "options": { "log": false } }] }
```

## Headless runs

`opencode run` returns when its first turn ends. The server keeps continuing the goal after that, so a headless goal needs a server that outlives the command. `opencode run --standalone` stops its private server on exit, and the goal stops with it; it comes back paused the next time a server starts.

For CI, start a server, create the goal through `opencode run --server`, and poll the goal until it leaves `active`:

```sh
opencode serve --hostname 127.0.0.1 --port 4096 &
id=$(opencode run --server http://127.0.0.1:4096 --format json "Set a goal: <objective>" | grep -o '"sessionID":"ses_[^"]*"' | head -1 | cut -d'"' -f4)
until opencode api --server http://127.0.0.1:4096 post "/api/rpc/goal/get?location[directory]=$PWD" \
  -d "{\"input\":{\"sessionID\":\"$id\"}}" | grep -qv '"status":"active"'; do sleep 10; done
```

## Known limitations

Replies to `/goal` and to the goal controls are queued as synthetic messages. When the session is idle they show up at the start of the next turn, not immediately.

Changing the budget of an existing goal takes the goal controls in the terminal UI, or the `goal/apply` RPC with a `Budget` action. The `/goal` syntax has no budget-only form; `/goal --tokens <n> <objective>` replaces the goal instead.

The subagent watchdog and the shell loop guard have these limits:

- The loop guard reads the `execute.before` hook. Shell calls made through CodeMode may skip that hook, so the guard may never see them. This is unverified.
- A tool call that never returns, such as a shell command with `timeout: 0`, is never interrupted, because the watchdog does not interrupt an open tool call.
- Subagents created before a server restart are not watched.
- A child that has its own goal gets that goal paused when the watchdog interrupts it, because the interrupt API takes no reason. The goal stays paused until the user resumes it.
- The watchdog and loop-guard canaries were verified on OpenCode v2.1.4. They run in the CI matrix, not yet run on 2.0.24.

## Attribution

The prompts in `templates/` and the tool descriptions are adapted from the `/goal` extension in [OpenAI Codex](https://github.com/openai/codex/tree/main/codex-rs/ext/goal), licensed under Apache-2.0. They were modified to use OpenCode tool names. See `NOTICE`.

## Develop

Requires Bun. Clone the repository and load your checkout instead of the published package:

```sh
git clone https://github.com/heliowap/opencode-v2-goal-plugin.git
cd opencode-v2-goal-plugin
bun install
opencode plugin remove github:heliowap/opencode-v2-goal-plugin   # if installed, so only one goal plugin loads
ln -sfn "$PWD/src" ~/.config/opencode/plugins/goal
```

The directory link loads both entrypoints: `src/index.ts` on the server and `src/tui.tsx` in the terminal. If you previously linked the single file `~/.config/opencode/plugins/goal.ts`, remove it and run `opencode service restart` before adding the directory link. OpenCode 2.0.24 fails to load a directory that replaces a single-file plugin of the same name in a running service, with `Cannot find package '@opencode/plugin'`.

OpenCode reloads the plugin when you save files under `src/`.

```sh
bun test                      # unit tests, 100% line and function coverage enforced
bun run typecheck
bun run mutation              # mutation testing: every mutant of the files in stryker.config.json must be killed
bun test ./host               # host canaries: needs `opencode` 2.0.24 on PATH and `tmux` for the terminal ones
LIVE_MODEL="<provider>/<model>" bun test ./host/live/live.test.ts   # live models, uses your provider config
```

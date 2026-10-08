# How Gas City works

A mental model of Gas City for engineers about to run it. It's built from the `gc` 1.5.0 docs and `--help` output, the Go source, and the `gascity` pack that a fresh 1.5.0 city pins. Where something comes from source alone, with no user doc behind it, it's marked **code-read**.

## The loop in one paragraph

A **city** is a directory of config that a background **supervisor** keeps running. You register a project with the city as a **rig**. Work is a **bead**, a record in a Dolt-backed store. When you **sling** a bead to an agent, the city's controller starts a Claude Code session (or another coding agent) in tmux. The agent claims the bead, does the work, and closes it. A **formula** turns one request into a graph of beads, each routed to a role. Every step is written to an event stream you can watch.

```
you ──gc sling──▶ bead ──▶ controller (inside the supervisor)
                               │ starts
                               ▼
                   agent session: claude in tmux, cwd = rig root
                               │ claims bead, works, closes bead
                               ▼
                         rig (your repo) + event stream
```

## The parts

**City.** A directory such as `~/my-city` holding `city.toml`, `pack.toml`, `agents/`, `formulas/`, `orders/` and runtime state in `.gc/`. `pack.toml` is the portable definition you could share. `city.toml` holds this machine's settings: providers, rigs and imports. `gc init <dir>` creates it.

**Supervisor.** One per-user service per machine; on macOS, the launchd job `com.gascity.supervisor`. It runs every registered city and serves the dashboard. `gc init` installs it at step 7 of 8, so if init stops earlier you have a city and no supervisor. The service file stores a snapshot of the PATH and environment of the shell that installed it, which is why the shell you run `gc init` from matters.

**Controller.** The loop inside each city. On every reconcile tick it starts and stops sessions to match the desired state, runs due orders, and advances formula graphs. If an agent dies, its bead stays open and a new agent picks it up. Gas City's architecture glossary calls it the controller, and the newer docs call it the orchestrator.

**Rig.** A project directory, usually a git repo, registered with `gc rig add <path>`. It gets its own bead ID prefix and agent scope. Isolation is by prefix: the city and all its rigs share one store. Git isn't required. `gc rig add` detects the default branch, writes `.beads/` into the repo, and adds `.beads/*` to its `.gitignore`.

**Bead.** One unit of work, with an ID such as `vi-12`, a title, a description, a status (`open`, `in_progress`, `closed`), dependencies and labels. Almost everything is a bead: tasks, mail between agents, workflow roots, and **convoys** (groups of beads tracked together). Beads live in Dolt, a SQL database with git-style versioning, and `bd` is their CLI. `gc bd ...` runs `bd` in the right rig.

**Sling.** `gc sling <target> "<text>"` creates a bead, attaches a workflow, and routes it to an agent. `gc sling <target> <bead-id> --on <formula>` attaches a formula to an existing bead. Plain-text sling uses the default formula `mol-do-work`: read the bead, do the work, close it. Agents don't ask before they start. Gas City's design rule is "If you find work on your hook, you run it."

**Formula.** A TOML recipe of steps with `needs` dependencies. Running one creates a root bead and one bead per step, and steps with no unmet `needs` run in parallel. `gc formula cook <name>` creates the beads without routing them, so you can inspect the graph before anything runs.

**Pack.** A bundle of agent definitions, formulas, prompts and orders. The default `gascity` template imports the `gascity` pack at city scope and its `roles` into every rig. That pack provides the build formulas (`build-basic`, `build-from-plan` and others) and the `gc.*` roles that run them: requirements planner, design author, task decomposer, implementation worker, reviewer, run operator and publisher. A fresh city pins the pack to the release embedded in the `gc` binary, not to whatever clone you have locally.

**Provider.** The word has three meanings. In an agent block it's the coding agent (`claude`, `codex` and so on). Under `[session]` it's where sessions run (tmux by default). For beads it's the storage backend (Dolt by default).

**Orders, mail, nudges.** An order runs a formula or script on a trigger such as an interval, a cron schedule or an event. Mail is a bead of type `message`. A nudge types text into a running session.

## What an agent actually reads

This is what decides whether an agent does what you meant. For a `build-basic` run on the pinned pack:

1. **Its role prompt.** Every `gc.*` role prompt opens by claiming its routed bead and printing it. The first work text the agent sees is that **step bead's title and description**. The role prompt says outright not to assume a parent or root bead describes its work.
2. **The step's instructions.** Each formula step has a description file in the pack (`assets/workflows/<formula>/<step>.md`). That text becomes the step bead's body (an inference from the source; I haven't confirmed it's copied verbatim).
3. **Your request, indirectly.** Your goal goes in the target bead you sling. The requirements step tells the agent to "preserve the input target", but no step prompt says exactly where to read the target bead's text from (code-read).
4. **A context bundle, if you pass one.** `--var context_path=<file>` names one YAML or JSON file that lists `items:`, each with only `name`, `path` and `description`. The pack's validator rejects directories, binary files, files over 1 MB, secret-looking paths, and anything outside the bundle's directory unless you allow it with `--allow-root`.
5. **Whatever is in the rig.** The agent's working directory is the rig root unless the role sets `work_dir`, and the `gc.*` roles don't (code-read). Claude Code therefore loads the rig's `CLAUDE.md` the way it would in any repo. `gc` doesn't inject it.
6. **Hooks.** `gc` launches Claude with `--settings <city>/.gc/settings.json`. That file holds `gc`'s SessionStart, PreCompact and UserPromptSubmit hooks, merged with anything you put in `<city>/.claude/settings.json` (code-read). It contains no PreToolUse hook unless you add one.

Two consequences follow:

- **No confinement.** No `gc` or pack mechanism limits an agent to one branch or directory. The docs describe `gc`'s commands as "a feature, not a sandbox". Agents run with `--dangerously-skip-permissions` by default. If you need a rule, state it in text the agent reads (the bead, a rig `CLAUDE.md`, a prompt patch) and enforce it outside Gas City.
- **Where the code ends up.** The implementation worker leaves the rig root for a per-item git worktree under `<rig>/worktrees/<id>`, but only because its prompt says to. On the pinned pack, that worktree is detached from the rig's current HEAD, and commits stay there. Publishing (`push`, `open_pr`) is off by default.

## Watching and stopping

| Need | Command |
|---|---|
| City overview | `gc status` |
| One bead, live | `gc bd show <id> --watch` |
| Sessions | `gc session list`, `peek <id> --lines N`, `logs <id> -f`, `attach <id>` (detach with `Ctrl-b d`) |
| Event stream | `gc events --follow` |
| Usage | `gc costs`, a list-price estimate from `.gc/usage.jsonl` with no budget or cap |
| Health | `gc doctor` |
| Pause | `gc suspend` / `gc resume`, or `gc rig suspend <rig>` |
| Stop | `gc stop`, which also unregisters the city from the supervisor |

Don't treat a successful launch as a finished workflow. Check the workflow root bead and the files under `artifact_root` (including `factory-run.md`) to see what actually happened.

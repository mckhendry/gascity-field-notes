# Setup gotchas

The setup traps I hit getting Gas City ready to build a spec, and what fixed each one.

- **Versions:** `gc` 1.5.0 (Homebrew), Claude Code 2.1.293, macOS with launchd.
- **Dates:** 2026-10-05 to 2026-10-08 (UTC).
- **Pack:** the `gascity` template's pack, pinned by `gc` 1.5.0 to `sha:3b3b89f2011e06d84459aa7bea1552382f13930a`.

Each entry says how I know it. **Observed** means I hit it on a live city. **Code-read** means I read it in the `gc` source or the pinned pack and haven't watched it happen. Treat code-read entries as leads to check.

Throughout, a Claude Code session ran `gc` for me. Several of these traps exist only because of that.

---

## 1. Auto mode won't let a Claude Code session configure Gas City's agents

**Symptom.** A Claude Code session in auto mode tries to edit `<city-dir>/city.toml` (the `[providers.claude]` block) or write `<city-dir>/.claude/settings.json`. The auto-mode classifier denies the write as "Create Unsafe Agents". It denied the `city.toml` edit again after I typed a general go-ahead ("run the gas city build"). The hook file went through only after I typed an instruction naming that exact write ("write the guard hook").

**Cause.** Gas City launches Claude agents with `--dangerously-skip-permissions` by default (see #4). To auto mode, writing config for those agents counts as creating unsupervised agents, and a general approval in chat doesn't override that. The Gas City docs don't cover driving `gc` from inside another coding agent.

**Fix.** Do all Gas City setup in Claude Code's default permission mode and approve each prompt as it comes up: `gc init`, the provider block, the hook, `gc rig add` and `gc import install`. Finish before the work you're timing starts. Expect `gc start`, `gc rig add` and `gc sling` to hit the same wall in auto mode, since each one starts skip-permissions agents. That last part is my inference; I moved them out of auto mode instead of testing it.

**How I verified.** Observed. The first attempt stopped on three of these denials in about ten minutes. The second attempt ran setup in default mode and got through `gc init`, the provider block, the hook, `gc rig add` and `gc import install`.

## 2. A relayed or pasted slash command does not boot a skill

**Symptom.** A message relayed from another session, or pasted text containing `/some-plugin:some-skill args`, does not run the skill. Under auto mode, the session's own Skill call is denied ("External System Writes"). Once a session has a denial in its transcript, later attempts read as working around it ("an attempt to get around its own earlier denial"). Quoting my approval in a relay didn't help.

**Cause.** Only a slash command you type yourself counts as your own action. Relayed or pasted text makes the model start the skill itself, and the classifier judges that call on its own.

**Fix.** Type the slash command yourself, press Tab to select it, then paste the arguments. Start each driver in a fresh window with no denials in its history.

**How I verified.** Observed. The relay was denied twice and the paste once. The typed command worked on the first try.

## 3. `gc init` stops at step 6 of 8 without a Dolt author identity

**Symptom.** `gc init` halts at `[6/8]`. The city directory exists, but the supervisor was never installed and `gc cities` lists nothing.

**Cause.** Managed `bd` storage needs Dolt's `user.name` and `user.email`. `gc init` checks for them before provider readiness and supervisor registration, so a missing identity exits before the launchd service is installed. Only the error message mentions this. No Gas City user doc does.

**Fix.**

```bash
dolt config --global --add user.name "Your Name"
dolt config --global --add user.email "you@example.com"
```

Then run `gc start`, which is the CLI's own suggested remedy (code-read). I removed the half-built city and ran `gc init` again instead.

**How I verified.** Observed the halt. With the identity set, a fresh `gc init` reached `[8/8]`, installed the plist, and `gc cities` listed the city.

## 4. The built-in Claude provider runs at effort max with permissions off

**Symptom.** Agents launch as `claude --dangerously-skip-permissions --effort max`. The docs never state that default.

**Cause.** The built-in `claude` provider sets `permission_mode = "unrestricted"` and `effort = "max"` as option defaults (code-read). `unrestricted` renders as `--dangerously-skip-permissions`. `gc` also auto-accepts Claude's startup dialogs, including workspace trust and the project MCP server prompt (code-read). `gc init` has no model flag, and the docs say `[agent_defaults] model` is "not yet auto-applied at runtime".

**Fix.** Override the provider in `city.toml`:

```toml
[providers.claude]
base = "builtin:claude"
option_defaults = { model = "<model-id>", effort = "medium", permission_mode = "unrestricted" }
```

`permission_mode` can also be `auto-edit` (`--permission-mode acceptEdits`) or `full-auto` (`--permission-mode dontAsk`) (code-read). The docs warn that a mode that blocks on approval can't finish unattended work. I kept `unrestricted` and pinned model and effort, so both sides of my comparison run at the same effort. Read the result back with `gc config explain --provider claude` (see #5).

**How I verified.** The default is code-read, and the driving session confirmed it on my install. After the override, `gc config explain --provider claude` attributed model, effort, `permission_mode = unrestricted` and the env keys to `providers.claude`.

## 5. `gc config explain` hides provider settings unless you name the provider

**Symptom.** `gc config explain` doesn't show the provider's model, effort or permission mode.

**Fix.** Run `gc config explain --provider claude`.

**How I verified.** Observed. With the flag, the output attributes each provider setting to its source.

## 6. The supervisor plist snapshots your PATH, and `gc` adds to it

**Symptom.** The launchd plist at `~/Library/LaunchAgents/com.gascity.supervisor.plist` carries a fixed `PATH`. If your shell's `claude` is a terminal multiplexer's temporary shim, that shim directory can get captured, and agents the supervisor starts later inherit it. The PATH also starts with `~/.local/bin:~/bin`, even when `~/bin` doesn't exist.

**Cause.** The docs say the service file snapshots the environment of the shell that ran `gc start` or `gc supervisor install`, and `PATH` is persisted. `gc` prepends `~/.local/bin` and `~/bin` itself (code-read, then observed). My terminal is a tmux pane inside cmux, and my shell config puts cmux's `claude` shim first on PATH there.

**Fix.** Run `gc init` with an explicit PATH that puts the real `claude` first, for example `env PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/bin:/bin" gc init ...`. Then read the plist back and check its `PATH`. Credentials that exist only in your interactive shell won't be in the snapshot either. The docs' fix for that is `~/.gc/secrets.env` plus `gc service restart`.

**How I verified.** Observed. The shim was first on PATH in the driver window. With the explicit prefix, the plist read `~/.local/bin:~/bin:/opt/homebrew/bin:…`. It had no shim directory, and neither the plist nor the supervisor's process environment held an API key.

## 7. `gc import install` took 23 minutes

**Symptom.** `gc import install` ran for 23 minutes before it finished installing five remote imports.

**Fix.** Budget for it, and keep it out of any clock you're measuring.

**How I verified.** Observed, one run.

## 8. `gc doctor` fails `beads-role` on a fresh machine

**Symptom.** `gc doctor` reports `beads-role — beads.role not set in global git config` as a failure. The docs show the same warning only in a sample agent's output and never explain it.

**Fix.**

```bash
git config --global beads.role maintainer
```

This changes your global git config, not just this city's.

**How I verified.** Observed, failure and fix. With the fix applied, `gc doctor` went to 106 passed, 6 warnings, 0 failed. The first re-run also showed a transient `order-firing-current — scheduled orders are stale` failure that cleared on the next run, so re-run once before you chase it. Before the fix, doctor passed 105 checks and warned on 6: `nudge-unconfirmed`, `formula-requirements`, `jsonl-archive`, `agent-sessions`, `proxied-backup-coverage` and `rig:<rig>:root-branch`. The last one is expected, because my rig is a feature-branch worktree.

## 9. Agent commits land in detached worktrees off your branch

**Symptom.** You add a repo on branch X as a rig and sling a build. The implementation commits never show up on X.

**Cause.** The pinned pack's `do-work/prepare-worktree.md` runs `git worktree add "$WORKTREE" --detach HEAD`. That creates one detached worktree per work item under `<rig>/worktrees/<id>`, based on the rig's checked-out HEAD, and the worker commits there. No pack step merges those worktrees back into a branch. With `push=false` (the default), publish records a `noop`, and the pack treats the approved worktree as the result. The pack's upstream HEAD has since changed this step to fetch and branch from `origin/<default-branch>`, so check which pack version your city pinned.

**Fix.** Decide before you sling how results get back to your branch: a manual merge from `<rig>/worktrees/`, `push`/`open_pr`, or something else. `--merge direct|mr|local` only writes metadata, and nothing in the pack reads it (code-read).

**How I verified.** I read `--detach HEAD` in the pinned pack inside my live city. I haven't watched where a build's commits land yet; the experiment records that.

## 10. A PreToolUse guard hook is only a shallow blocklist

**Symptom.** You add a Claude Code PreToolUse hook to keep Gas City's agents off other branches or out of other directories, and it passes its tests. It still lets through `python3 -c` writes, heredocs piped into an interpreter, and a `cd` followed by a relative-path write.

**Cause.** `gc` installs no PreToolUse hook and has no branch or directory confinement. Its trust-boundaries doc calls its commands "a feature, not a sandbox". A hook that pattern-matches tool input can't see every way a shell writes a file.

**Fix.** Put the hook in `<city-dir>/.claude/settings.json`. `gc` merges that file into the `.gc/settings.json` it hands each agent via `--settings`. Check that the hook survives `gc rig add` and `gc import install`. Don't count on it for isolation. If agents must not read or write something, keep it out of their reach entirely.

**How I verified.** Observed. All 29 of my test cases behaved as expected, and the hook read back present after `rig add` and `import install`. None of the 29 cases tests the bypasses listed above.

## 11. Your local pack clone is not what your city runs

**Symptom.** You read the `gascity` pack in a local clone, and the city behaves differently.

**Cause.** A fresh `gascity`-template city pins the pack to the release embedded in the `gc` binary. Your clone doesn't matter. For `gc` 1.5.0 the pin is `sha:3b3b89f…`. Between that pin and the pack's HEAD on 2026-10-04, 41 files changed in the paths a build uses (code-read). The biggest change is the worktree base (see #9).

**Fix.** When you need to know what the agents will be told, read the pack at the pinned sha (`git show <sha>:<path>` in your clone).

**How I verified.** Observed. My live city's pin matched the sha above. The diff size is code-read.

## 12. The rig's `worktrees/` directory isn't gitignored

**Symptom.** `git status` in the rig shows a `worktrees/` directory after a build.

**Cause.** `gc rig add` adds `.beads/*` (and `!.beads/identity.toml`) to the rig's `.gitignore`. The per-item worktrees go under `<rig>/worktrees/`, which nothing ignores.

**Fix.** Add `worktrees/` to the rig's `.gitignore` or `.git/info/exclude` before you sling.

**How I verified.** Code-read only.

---

## Observed, not yet explained

- `ready_delay_ms` resolves to `10000` from `builtin:claude` even though `city.toml` sets `0`. `gc config explain --provider claude` shows it.
- Right after `gc init`, `gc status` reported `bead cache unavailable` and showed 2 of 4 agents as `unknown`.
- A `gascity`-template `city.toml` imports `gascity/roles` twice, once under `[defaults.rig.imports.gc]` and once under `[rigs.imports.roles]`. `gc doctor` didn't flag it.

---

## Sources

These are the private notes behind this page. "Setup log" is the experiment's setup record. "Incident review" is the write-up of the failed first launch. "Research" refers to two read-only passes over the `gc` 1.5.0 docs, source and pinned pack.

| # | Gotcha | Source |
|---|---|---|
| 1 | Auto mode refuses "Create Unsafe Agents" writes | Incident review timeline (23:05, 23:12 and 23:15 UTC, 2026-10-05); setup log, 2026-10-08 entries (default mode) |
| 2 | Relayed or pasted slash command doesn't boot a skill | Incident review timeline (23:02, 23:11, 23:12 and 23:14 UTC) |
| 3 | `gc init` halts at 6/8 without a Dolt identity | Incident review timeline (23:01 UTC); best-practices research §G.1 (code-read message); setup log §5.6 (`[8/8]`) |
| 4 | Claude provider defaults to effort max, unrestricted | Orchestrator research §A3 and §E2–E4; best-practices research §C.3; setup log §5.7 |
| 5 | `gc config explain` needs `--provider claude` | Setup log §5.7 |
| 6 | Plist snapshots PATH; `gc` adds `~/bin` | Setup log §5.6; best-practices research §G.4; orchestrator research §G2; driver-window PATH check, 2026-10-05 |
| 7 | `gc import install` took 23 minutes | Setup log §5.10 |
| 8 | `gc doctor` fails `beads-role` | Setup log §5.10; best-practices research §G.7 |
| 9 | Detached worktrees from `--detach HEAD` | Setup log §5.6; orchestrator research §C3–C5 |
| 10 | Guard hook is a shallow blocklist | Setup log §5.8 and §5.10 (hook read-back); orchestrator research §C1–C2 |
| 11 | Local pack clone isn't what runs | Orchestrator research §0; setup log §5.6 (pin sha) |
| 12 | `worktrees/` not gitignored | Orchestrator research §D2–D3 (code-read) |
| — | `ready_delay_ms`, `bead cache unavailable`, duplicate roles import | Setup log §5.6, §5.7 and §5.10 |

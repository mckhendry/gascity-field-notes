# Setup gotchas

The traps I hit getting Gas City ready to build a spec and then running that build, with what fixed each one. Entries 1–16 are setup. Entries 17–26 came from the first run's two milestones. Entries 27–37 came from the reconfigured run, its recovery and the teardown ([results](../experiments/two-driver-results.md#run-c-gas-city-reconfigured)).

- **Versions:** `gc` 1.5.0 (Homebrew), Claude Code 2.1.293 at setup and 2.1.296 from the first run's milestone 2 on (it auto-updated mid-run), macOS with launchd.
- **Dates:** 2026-10-05 to 2026-10-15 (UTC).
- **Pack:** the `gascity` template's pack, pinned by `gc` 1.5.0 to `sha:3b3b89f2011e06d84459aa7bea1552382f13930a`.
- **Pack for entries 27–37:** pre-release pack main, the first commit that fixes the check-script paths ([what-the-docs-say.md](what-the-docs-say.md#checks)), with overrides in my own fork of the pack.

Each entry says how I know it. **Observed** means I hit it on a live city. **Code-read** means I read it in the `gc` source or the pinned pack and haven't watched it happen. Treat code-read entries as leads to check.

Entries 27–37 also carry a **class**: a **Gas City defect**, **configuration** (how Gas City is meant to be set up, or a design you have to plan around), or **my mistake** (something I set up wrong or should have caught).

Throughout, a Claude Code session ran `gc` for me. Several of these traps exist only because of that.

---

## 1. A Claude Code session in auto mode won't configure Gas City's agents

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

**How I verified.** The default is code-read, and the session running `gc` confirmed it on my install. After the override, `gc config explain --provider claude` attributed model, effort, `permission_mode = unrestricted` and the env keys to `providers.claude`.

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

**How I verified.** I read `--detach HEAD` in the pinned pack inside my live city, then watched it happen in two real runs: every item's commits stayed in detached worktrees. #18 covers what that does to delivery.

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

## 12. Neither `.gc/` nor the rig's `worktrees/` is ignored

**Symptom.** `git status` in the rig shows `?? .gc/` once agents have run there, and `?? worktrees/` after a build. Any agent step that runs a broad `git add` would commit them, and with `push` on, publish could push them.

**Cause.** `gc rig add` adds `.beads/*` (and `!.beads/identity.toml`) to the rig's `.gitignore`. Nothing ignores `.gc/` in the rig or the per-item worktrees under `<rig>/worktrees/`.

**Fix.** Before you sling, add `/.gc/` and `/worktrees/` to the repository's `.git/info/exclude`. That keeps them out of git without a tracked commit on the branch you're measuring. If the rig is itself a git worktree, the exclude file lives in the main checkout's `.git/` and applies to every worktree of that repo.

**How I verified.** Observed. A one-bead smoke test left `?? .gc/` in the rig, and the milestone-1 build left `?? worktrees/`. After both entries went into `.git/info/exclude`, `.gc/` no longer showed in `git status`, and `git check-ignore` confirmed `worktrees/`.

## 13. `gc rig add` commits to your rig's branch, then leaves it dirty

**Symptom.** After `gc rig add`, the rig's branch has a commit you didn't make: "bd init: initialize beads issue tracking". It adds `.beads/.gitignore`, `.beads/README.md`, `.beads/config.yaml`, `.beads/metadata.json` and seven lines of `.gitignore` (the entries in #12). `git status` then shows `.beads/config.yaml` as modified.

**Cause.** Rig add runs `bd init`, which commits its files to whatever branch the rig has checked out. `gc` then appends `dolt:` / `shared-server: false` to `.beads/config.yaml`, a file that commit has just started tracking.

**Fix.** Keep the commit. Reverting it deletes the bead store's config and metadata from the rig. Hide the later edit from git instead:

```bash
git -C <rig> update-index --skip-worktree .beads/config.yaml
```

If you measure agent output by commits on the rig's branch, start counting after this commit and any setup commits of your own, not from the branch's base.

**How I verified.** Observed. I found the commit in the rig branch's log after rig add, and before it `.beads/config.yaml` wasn't tracked on my base branch. With the flag set, `git status` showed only my own `.gitignore` edit.

## 14. Context-bundle paths resolve from the bundle's directory, not the rig root

**Symptom.** The pack's context-bundle validator (`assets/scripts/validate_context_bundle.py`) exits 1 on `<rig>/<dir>/context.yaml` when its items are written from the rig root, like `path: <dir>/spec.md`. Each item resolves to `<rig>/<dir>/<dir>/spec.md`.

**Cause.** The validator resolves a relative `path` against the bundle file's own directory (`resolve_item_path` in the pinned pack). `--allow-root` only bounds where paths may resolve to. It doesn't change the base.

**Fix.** For files next to the bundle, use bare filenames (`path: spec.md`). Write anything else relative to the bundle's directory, or as an absolute path. Run the validator before you sling:

```bash
python3 <pack>/assets/scripts/validate_context_bundle.py <rig>/<dir>/context.yaml --allow-root <rig>
```

**How I verified.** Observed. Root-relative paths failed with every prefix doubled. With bare filenames the validator returned `{"items": 3, "ok": true}`. I then read the resolution rule in the pinned pack.

## 15. `gc rig add --include` of the roles pack leaves formulas with no agents

**Symptom.** You add the rig with `--include` pointing at the `gascity/roles` pack. Slinging a formula such as `build-basic` then fails "agent not found", because the rig's agents are named `roles.*`.

**Cause.** The `gascity` template already imports `gascity/roles` for every rig under the name `gc` (`[defaults.rig.imports.gc]`). `--include` writes `[rigs.imports.roles]`, which displaces that default instead of adding to it. Formulas route to `gc.*` roles (`build-basic` names eleven), and none of those exist on the rig. This is also why my `city.toml` seemed to import the roles pack twice.

**Fix.** Add the rig with no `--include`. On a rig already added that way, deleting `[rigs.imports.roles]` isn't enough: the default applies only at rig add, so the rig is left with no role agents at all. Add a `[rigs.imports.gc]` block with the same source and pin as `[defaults.rig.imports.gc]`, then run `gc import install`. `gc config explain` should then list `<rig>/gc.*` roles.

**How I verified.** Observed. The failure came first. After the repair, `gc config explain` listed twelve `<rig>/gc.*` roles and the sling dry run resolved (see #16). A dry run proves only the formula's entry point. Its handoffs to the other `gc.*` roles first run on a real bead, and I haven't watched that yet.

## 16. The sling target needs its rig prefix, and `--dry-run` hides your vars

**Symptom.** A sling aimed at the bare `gc.run-operator` fails "agent not found". A `gc sling --dry-run` with the right target prints the plan but none of the `--var` values you passed, so it can't confirm them.

**Fix.** Write the target as `<rig>/gc.run-operator`. Check each `--var` name against the formula's declared vars in `gc formula show <formula> --json`.

**How I verified.** Observed. The dry run on `<rig>/gc.run-operator` named the formula, resolved onto the rig, planned `gc formula cook <formula> --attach <bead-id>` and started no session. None of my four vars appeared in its output, and all four are declared in `formula show`.

---

## 17. Workflow roots get routed to implementation workers

**Symptom.** During the implementation drain, implementation workers claim a bead that turns out to be a do-work workflow root (`gc.kind=workflow`), not a work item. Each one releases it, and about a minute later another worker claims it. Some workers stop and ask a human instead: release it and drain, or close it as failed? In milestone 2, three workers stopped like this, and `gc` flagged each step as stalled.

**Cause.** An upstream bug, open as [gastownhall/gascity#6461](https://github.com/gastownhall/gascity/issues/6461). It was reported against 1.4.1, and the code is unchanged in 1.5.0 (code-read at tag `v1.5.0`):

- The drain decorates each item's do-work recipe with the drain's execution route, and `DecorateGraphWorkflowRecipeWithDefaultBinding` stamps `gc.routed_to` on the root unconditionally (`internal/graphroute/graphroute.go:645-652`). With the pack's default `implementation_target`, that's `<rig>/gc.implementation-worker`.
- The compiler links the root to `workflow-finalize` with a `tracks` edge, not `blocks`, and its comment says the root is a latch that is "never routed or dispatched" (`internal/formula/compile.go:680-697`). So once the previous drain item closes, the root is ready.
- The worker's work query, `bd ready --metadata-field gc.routed_to=<target> --unassigned …`, doesn't filter on `gc.kind=workflow` (`internal/config/workquery.go:226-228`). The claim check's first branch accepts any bead whose `gc.routed_to` matches the worker (`cmd/gc/cmd_hook_claim.go:3103-3125`).
- The pack's worker prompt says workflow and scope beads are latches it "should not receive", but gives no instruction for what to do with one. So some workers ask.

This came from the default template and pack. A hand repair of my rig import (#15) restored the default binding byte for byte and doesn't appear anywhere in the chain.

**Fix or workaround.** None in 1.5.0. The roots end up closed by workflow cleanup, so the run keeps going, but the churn burns seats (see #22), and a worker that asks a human stalls until someone answers or its lease runs out (see #20). My untested guess: telling workers outright to release a latch bead and drain would turn the questions into silent releases.

**How I verified.** Observed, then code-read. All 15 do-work roots under the milestone-2 drain carried `gc.routed_to=<rig>/gc.implementation-worker` and `gc.kind=workflow`. Each named root's assignee changed to a new worker session about once a minute, across roughly ten sessions per root. The other twelve roots went through the same cycle, but their workers released them silently. So the count of three prompts reflects how each worker judged the bead, not how many roots were affected. A read-only trace then followed the chain through the `v1.5.0` source and the pinned pack.

## 18. Finalize leaves the work detached, and publish can't deliver it

**Symptom.** The build finishes and finalize writes `factory-run.md`, but the rig's branch hasn't moved. Turning on `push` and `open_pr` doesn't change that. In my milestone-1 run (`push` off), all 11 build commits stayed on a detached HEAD in one item worktree. Before milestone 2, a publish run with `push=true` and `open_pr=true` closed `pass` and moved nothing. In milestone 2 (`push` and `open_pr` on from the start), publish closed `fail` with "Nothing approved to push or PR", and the branch on the remote was where it started.

**Cause.** Three things stack:

- Each item gets a detached worktree based on the rig's HEAD at that moment, not on its dependencies' commits (#9). Agents pull in dependencies by hand. In milestone 1 they cherry-picked, so the same change ended up under two SHAs. In milestone 2 they merged or stacked on the original SHAs. The last item's worktree ended up holding the integrated line both times.
- Finalize doesn't merge that line onto the branch. It hands the merge to a human: the smoke test's report named `git merge --ff-only <sha>` as the next action, and milestone 2's report listed "operator integration" of the final commit.
- Publish pushes the branch's current head. It can't push commits that sit on a detached HEAD in an item worktree. It also skips when the review never approved: in milestone 2 the review loop failed out (#19), and the artifact check on finalize failed on PyYAML (#21), so publish had nothing approved to send.

**Fix.** Plan your own integration step. Find the item worktree whose HEAD holds every item's commits, check that it does, then fast-forward the branch to it and push. If you run a guard hook, check that it allows publish's push. Mine blocked the lease-checked push on the one run where publish tried.

**How I verified.** Observed across a smoke test and both milestones. The one fast-forward I did by hand, from the setup commits to the milestone-1 tip, worked cleanly and touched no `.gc/`, `worktrees/` or `.beads/` paths.

## 19. The review loop can't converge on decisions it isn't allowed to make

**Symptom.** In autonomous mode, the review stage keeps iterating with no new commits. Each pass ends in "iterate pending operator items" and takes about five minutes, until the loop's check gives up and closes `fail`. Finalize then reports `blocked`.

**Cause.** The review routes two kinds of item to a human: spending approvals (in milestone 2, about $1 to resume three ingests that timed out, plus a paid eval rerun on one input) and decisions about the bar (fix the test harness or pick a different bar, and whether some evidence is enough). The pack gives the loop no spend budget and no agent with authority to rule on either, and `interaction_mode=autonomous` doesn't change that. So the loop re-reviews the same open items. In milestone 2 it ran six iterations, four of them no-ops, before its control closed `fail`. That's the review loop check's `max_attempts` of 6, not the formula's `max_iterations` of 10. Milestone 1 failed the opposite way: the fixer left "a gate decision and a rerun" to me, the review approved on its third pass while the gate was still failing, and a prompt change landed after the gate ran without the gate running again.

**Workaround.** Untested, but: put a spend ceiling and a decision owner in the goal bead before you sling. Say what the agents may spend without asking and who, or which agent, rules on questions about the bar. In my comparison, the other driver had an advisor agent that ruled on this class of question during the run, and its review didn't stall this way.

**How I verified.** Observed in both milestones, from the review synthesis and fix-pass close reasons and from the finalize report.

## 20. Agents in autonomous mode stop to ask a human, and nothing tells you

**Symptom.** A step stalls. `gc session peek` shows the agent sitting at an interactive question (Claude Code's multiple-choice prompt) or a free-text question addressed to a person. Nobody is at that terminal, and `gc` sends no notification.

**Cause.** `interaction_mode=autonomous` doesn't stop an agent from asking. It happened four times across two milestones. Once, a run operator on its last check attempt asked how to handle a missing dependency (#21). The other three were workers holding misrouted roots (#17). One of the offered answers was to install packages into the system Pythons, a machine change outside anything I had allowed. The only way the question reaches anyone is if someone happens to be watching that session.

**Workaround.** Watch `gc events --follow` for `execution.step_stalled`, then `gc session peek <session>` the stalled step's session. To answer, `gc session attach <session>`. Read every option before you pick one. One of mine offered to install packages into the system Pythons. `gc` flags a stall by step age, so expect the flag a few minutes after the question appears.

**How I verified.** Observed. The milestone-1 question blocked the run for 52 minutes, until I attached and answered it. The three in milestone 2 got no answer. Their beads closed `fail` on their own, within about 5 to 20 minutes for two of them and later for the third.

## 21. Artifact checks fail on a PyYAML the controller can't see

**Symptom.** Every `build-artifact-valid.sh` control closes `gc.outcome: fail` after three attempts, each with `error: PyYAML is required to parse build artifacts`. Meanwhile the agent's own step closes `pass`, and it says the artifact is valid when it runs the validator itself. Failed checks don't block the workflow, so each stage just pays for about two extra agent attempts. In milestone 2 every retry also left an extra summary commit in the item worktree.

**Cause.** A fresh 1.5.0 city has no check scripts at all ([gastownhall/gascity#4239](https://github.com/gastownhall/gascity/issues/4239)). Without them, steps whose work passed close `control_quarantined`. I copied the pinned pack's scripts into `<rig>/.gc/scripts/` before the build. The scripts need PyYAML, and the controller's check doesn't find it. In my shell, `python3` resolved to a python.org install that had PyYAML. `/usr/local/bin/python3` and `/usr/bin/python3` didn't have it. The supervisor's PATH looked the same as my shell's, so the check most likely runs with a narrower environment. That last part is my inference. An open issue upstream, gascity-packs#340, reports the same symptom: PyYAML invisible to the gate, so the failure looks like a bad artifact.

**Fix.** Make PyYAML importable by every `python3` the controller might resolve. For me that means `/usr/local/bin/python3` and `/usr/bin/python3`, not just the one first on your PATH. Then run a single trivial bead and check that its artifact control closes `pass`. I didn't apply this during either milestone, so I haven't confirmed it.

**How I verified.** Observed in both milestones: every artifact control failed this way. Which interpreters had PyYAML I checked by hand. Where the check actually runs is inferred.

## 22. Idle worker slots wake and drain about every 50 seconds

**Symptom.** While one long item runs alone, implementation-worker sessions keep starting, finding nothing, acknowledging a drain and stopping, about every 50 seconds. Each wake is a fresh Claude session on your configured model. `gc costs` shows many sessions with one to three invocations.

**Cause.** The pool keeps scaling up on routed demand that no worker can actually use. The trace behind #17 ties at least part of this churn to the misrouted root: each worker claims it, releases it, and the pool starts another.

**Fix.** None in 1.5.0. Budget for it as real spend.

**How I verified.** Observed and measured. Over about 12 minutes of milestone 1, with one item running, worker slots woke and drained about 16 times. Counted from the transcripts at list price, 165 agent sessions that woke, found no work and stopped (449 calls, no writes, at most three tool calls each) cost $34.95 of Gas City's $144.40 for milestone 1. That count is conservative: sessions that made more than three tool calls before draining are counted as working. On a subscription, the same churn eats into the shared usage limit (#26).

## 23. The always-on mayor keeps the settings it started with

**Symptom.** After you configure the provider, new agents launch with your model, effort and environment. The `mayor` session doesn't. Mine ran `--dangerously-skip-permissions` on my real Claude profile, without the separate profile and isolation variables every other agent got.

**Cause.** The mayor starts at `gc init`, before you've written the `[providers.claude]` block (#4), and keeps running. Nothing restarted it when the provider config changed.

**Fix.** After the provider is configured, restart the mayor session. Then read its environment back with `ps -E` and check for your profile directory and isolation variables.

**How I verified.** Observed. A smoke test caught it. After a restart, `ps -E` showed the mayor on the separate profile.

## 24. The control dispatcher restarts itself, and workers change Claude session ids

**Symptom.** `gc session list` shows a new control-dispatcher session partway through a run. Separately, a stall watcher that follows a worker's transcript file fires while the worker is fine.

**Cause.** In milestone 1, the control dispatcher restarted twice, each time 15 to 40 seconds after a `session.work_query_failed` event (`bd --readonly --sandbox ready …` exited 1). Nobody intervened, and the work carried on. A worker can also move to a new Claude session id while keeping the same `gc` session and bead claim, so its transcript moves to a new file.

**Workaround.** Count dispatcher restarts, but don't chase them unless work stops. Key any stall watcher to the `gc` session and look up its current transcript on every check, rather than watching one file. `gc`'s own `execution.step_stalled` fires on step age, not transcript silence, so the two measures disagree. Pick one before the run.

**How I verified.** Observed. Both dispatcher restarts followed a failed work query. In milestone 2 my watcher on the old transcript raised a false stall. The worker was waiting on its own background batch under a new session id.

## 25. `claude plugin eval` can't run a package that shells out to `claude -p` (not Gas City-specific)

**Symptom.** A plugin's eval cases that drive a CLI calling `claude -p` all fail. The milestone-2 build's 8 cases scored 0 of 8.

**Cause.** `claude plugin eval` runs each case in an OS sandbox. Inside it, the home directory is unreadable, writes are confined to the workspace, and the case couldn't reach the graph database through the sandbox's proxy. The agent found no supported setting to widen access. Either driver could hit this. It comes from Claude Code's sandbox.

**Fix.** None found. Design the eval cases around the sandbox, or gate that layer some other way. The other driver's 8 eval cases for the same spec passed, so how the cases are written matters. I haven't compared the two suites.

**How I verified.** Observed on Claude Code 2.1.296: `casesPassed 0`, exit 1. The sandbox limits come from the building agent's report in its transcript, which I didn't reproduce.

## 26. A shared subscription usage limit pauses every agent at once (not Gas City-specific)

**Symptom.** Every agent stops partway through a step. A session shows "You've hit your session limit · resets <time>", then later "Usage limit reset · continuing automatically", and the work resumes by itself.

**Cause.** Every Gas City agent ran on the same subscription as the session driving it. Parallel workers plus idle churn (#22) used up the limit. This applies to any setup that puts many agents on one subscription.

**Fix.** Nothing needs restarting. Budget for the wait, and take it out of any wall clock you're comparing. Before you call a gap a stall, check whether the machine slept (`pmset -g log` on macOS). Lid-closed sleep accounted for part of both long gaps in my milestone-1 run.

**How I verified.** Observed. The limit message appeared in a run-operator session, and the agents resumed after the reset without a restart. It took 88 minutes off milestone 1's clock. It hit only the Gas City side because the other driver had already stopped.

---

## 27. The workflow-root claim race survives the #6461 workaround

**Symptom.** The reconfigured run used the issue thread's workaround for #17: the item's prepare step removes `gc.routed_to` from its workflow root. Implementation workers still claimed roots. The claimer declined the root as a latch bead and sat idle holding it, and `gc` raised `execution.step_stalled` 3 to 5 minutes later. Another worker did the real work meanwhile.

**Cause.** The same upstream bug as #17, [gastownhall/gascity#6461](https://github.com/gastownhall/gascity/issues/6461). Claims landed 3 to 25 seconds after the previous unit closed, before the next item's prepare step ran, so a workaround placed at prepare can't remove the route in time.

**Fix.** None in 1.5.0. The workaround still limits the damage. Once the route is gone, the orphan sweep reopens the root, nobody claims it again, and the item's workflow-finalize closes it whatever its status (code-read). So leave the root alone. Each race costs one idle session for a few minutes and blocks no work.

**How I verified.** Observed in every phase: 5 races in 8 items at milestone 1 (4 of them stalled), 3 stalls at milestone 2, 3 in the recovery and 1 in the third attempt. Every root closed `pass` with its item, and none needed a human. The finalize behavior was read in the `gc` 1.5.0 source before I relied on it.

**Class.** Gas City defect. The workaround lives in my pack fork.

## 28. The execution backstop kills an active worker after about 3 minutes of quiet pane

**Symptom.** A worker starts a long job (a batch, a gate run) in a background shell and ends its turn to wait. About 3 minutes later `gc` raises `execution.step_stalled`, kills the worker's tmux session and re-dispatches the step to a fresh process with no context. The background job dies with the session.

**Cause.** `gc`'s execution backstop treats about 3 minutes of quiet tmux pane on a pool worker holding a claim as a stall. With no `nudge` configured it escalates at once, drains the session, and assigned work can't cancel the drain. A background shell doesn't count as pane activity. No config key turns the backstop off, and a nudge only delays the drain, because pane activity never resets its attempt count (code-read in the `gc` 1.5.0 source). Upstream: [gastownhall/gascity#6748](https://github.com/gastownhall/gascity/issues/6748), where another city reports 267 drains in about 21 hours.

**Fix.** Shadow every live step prompt with a rule: never end a turn while a background job runs; wait in the foreground, or start the job under `nohup` with output to a file and check it in short foreground steps. I added that text to all 12 live step prompts in my fork before milestone 2. After that, one worker ran a 3 h 11 m batch in a single turn without a kill.

**How I verified.** Observed twice at milestone 1: a live batch and, later, a gate rerun were killed, and `gc`'s own events show the kill. The mechanism is code-read. It seemed to fire once per claim, since a re-dispatched claim of the same bead didn't fire again, but that's two observations, not proof. With the shadow in place there were no kills of active work across milestone 2, the recovery and the third attempt. An early reading put the threshold at 90 seconds; I corrected it to about 3 minutes from the source and the observed timing.

**Class.** Gas City defect.

## 29. Host sleep cuts an API call mid-turn, and the backstop recovers the idle claim

**Symptom.** A step stalls. The agent's pane shows "API Error: Your computer went to sleep mid-response. The response above may be incomplete." Its turn ended there, idle, still holding the claim. A few minutes later `gc` kills the session, re-dispatches the step, and the step is redone.

**Cause.** The Mac slept. I had waived keeping it awake on the belief that AC power was enough. It wasn't: the AC power profile also had a 1-minute sleep. Agents ran `caffeinate` on their own, and Claude Code holds a short `caffeinate -i` while busy, but neither held a closed lid on battery or woke a Mac already in maintenance sleep.

**Fix.** Keep the host awake for the whole run. Check `pmset -g` for the sleep setting on both AC and battery, run `caffeinate -dimsu` for the run's length, and keep the lid open. Once the host has slept, the backstop's recovery is the right behavior: the step reruns from scratch, which costs only that step.

**How I verified.** Observed twice: at milestone 2 (about 3 minutes redone) and in the recovery (about 5 minutes redone). Sleep waits from `pmset -g log`: 29 m 23 s at milestone 1, 69 m 33 s in four spans at milestone 2, and 29 m 35 s in the recovery.

**Class.** My mistake. The backstop's recovery worked as designed here.

## 30. The final report lands untracked where an item commits, so integrate collides

**Symptom.** Integrate fails: "git would overwrite untracked plugins/…/build-recovery/factory-run.md". Seven of eight items merged locally, the last didn't, and publish refused with `no_integrated_commit`. Nothing was pushed.

**Cause.** `build-basic` writes its run records, including the build-level final report at `gc.build.final_report_path`, untracked into the rig checkout under `artifact_root`. The gate item committed its own record at that same path on its item branch. Integrate merges in the rig checkout, where the untracked file blocks the merge. A related layout trap: stage artifacts (requirements, plan) sit untracked in the rig checkout, so item worktrees don't contain them, and agents read them by absolute path.

**Fix.** Before each run, copy the previous run's records out of the rig checkout and clear the artifact root, or use a fresh `artifact_root` per run. Tell items not to commit at the final report's path, or point `final_report_path` outside the checkout. After a collision: copy the file out, remove only the colliding file, and reopen integrate and publish (#31).

**How I verified.** Observed in the recovery. At prep I copied the previous run's records out but didn't clear the checkout, though the risk had been named. Finalize saw the collision and wrote its own report under a different name. For the third attempt the path was cleared, and integrate passed.

**Class.** Gas City defect in the build layout, plus my mistake: the risk was known before launch.

## 31. There's no per-step retry, so reopen the failed step's bead

**Symptom.** A step fails for a reason you've since fixed, such as #30. There's no command to rerun just that step, and a re-sling pours the whole formula again.

**Cause.** In `gc` 1.5.0, step state lives in beads, and I found no step-retry command. A re-sling reruns every stage from requirements on. For the recovery that would have cost about $100 and likely conflicted with the existing item branches.

**Fix.** Workaround: set the failed step's bead back to open with `bd`. Pool dispatch claimed it within about 30 seconds, even though the workflow root was already closed, and Gas City's own agents ran the step. Reopen one step at a time and wait for each to close. The root stays recorded `fail`, and the reopened bead keeps stale metadata from its first attempt (here, the old publish reason). Label any delivery made this way as human-assisted.

**How I verified.** Observed once, in the recovery: integrate was reopened and passed, then publish was reopened and pushed. I haven't checked whether a newer `gc` has a retry.

**Class.** Gas City defect (a missing feature).

## 32. Gas City's own usage figure undercounts, so price the cap from transcripts

**Symptom.** Gas City's usage figure read far below a transcript-priced count of the same window: $24.63 against $94.14 at milestone 1, $37.45 against $109.19 at milestone 2, $27.53 against $98.73 in the recovery and $6.42 against $31.64 in the third attempt, 2.9× to 4.9× low. It also lags: during one long session it stayed flat for 10 minutes, then caught up when the session exited. `gc costs` printed "No usage facts recorded yet" throughout.

**Cause.** Two upstream bugs: pool sessions that exit through `drain-ack` send only a few usage facts ([gastownhall/gascity#6672](https://github.com/gastownhall/gascity/issues/6672)), and usage rows are flushed on the next wake ([gastownhall/gascity#5825](https://github.com/gastownhall/gascity/issues/5825)). Two more things widen the gap:

- `gc` 1.5.0's model usage facts carry no city field, so my exec usage sink wrote them to an unknown-city file, not the city's.
- The transcript count includes the driver session watching the run, which Gas City can't see. So part of the gap isn't undercounting.

`gc costs` goes blank because an exec usage sink replaces the file it reads.

**Fix.** Enforce any spend cap from transcripts. My collector priced every agent transcript every 5 minutes and would have called `gc suspend` at $200; Gas City's figure was recorded beside it. No run reached the cap.

**How I verified.** Observed in all four runs. The lag matches #5825's shape.

**Class.** Gas City defect.

## 33. An in-run advisor rewrote history below the target branch's tip, so integrate failed

**Symptom.** Integrate fails with add/add conflicts in six files that no two items should both add. Seven items merge and the last doesn't, so nothing is delivered.

**Cause.** My advisor step ruled that a tracked `.beads/` directory, present on the branch from a setup commit, should come out of history. The fix worker rewrote every commit down past the target branch's tip, so that tip was no longer an ancestor of the item branch. The other items' branches were on the old history, and the rewritten branch carried the same files under new commits. The rewrite also removed the rig's tracked `bd` config from the branch, so before milestone 2 I had to restore those files on disk as ignored files for `bd` to reach the bead store.

**Fix.** Tell any advisor or fix step that commits reachable from the target branch are fixed and must never be rewritten. Remove an unwanted tracked file in a new commit. A `git merge-tree` dry run of the item branches against the target predicts this before integrate runs.

**How I verified.** Observed at milestone 1. A read-only `git merge-tree` forecast named the six conflicting files before integrate ran, and integrate failed on exactly those.

**Class.** My mistake: my advisor override had no rule protecting the target branch. The ruling itself was the agent's judgment.

## 34. `gc stop` unregisters the city, and Dolt restarts on any later `bd` read

**Symptom.** After `gc stop`, `gc cities` prints "No cities registered". Separately, Dolt servers came back right after the stops.

**Cause.** Both are documented. `gc stop` also unregisters the city. Under the default proxied Dolt, any `bd` read starts Dolt again ([what-the-docs-say.md](what-the-docs-say.md#teardown-and-a-second-city)).

**Fix.** Tear down in this order: `gc stop` on each city, then `gc supervisor stop --wait`, then look for leftover Dolt processes and stop them. Run no `bd` command afterward. Write each city's path down first, because `gc cities` won't list it once stopped.

**How I verified.** Observed at teardown on 2026-10-15: "Unregistered city … City stopped", then "No cities registered". Two Dolt servers restarted right after the stops and had to be stopped by signal. Nothing was deleted.

**Class.** Configuration (documented behavior).

## 35. The supervisor is a launchd job, so closing your sessions stops nothing

**Symptom.** I closed every Claude Code session tied to the run, but both cities kept running: the mayors stayed up and the supervisor process stayed alive. Left idle overnight, one city logged 9,472 events, all of them its own patrols (orders firing, beads maintenance cycles), and no work.

**Cause.** The supervisor is a per-user launchd job, `com.gascity.supervisor`, that runs every registered city regardless of any terminal or session ([how-gas-city-works.md](../concepts/how-gas-city-works.md#the-parts)).

**Fix.** Stop it explicitly (#34). `gc supervisor stop --wait` leaves the launchd job disabled with its plist untouched. It comes back on `gc supervisor install` or `gc start`.

**How I verified.** Observed: the cities were still up after the sessions closed, and the overnight event count came from `gc events`.

**Class.** Configuration (by design). My expectation was wrong.

## 36. Agents can edit what grades them

**Symptom.** A worker whose gate is failing edits the product's scoring prompt so claim validity is scored without web verification, though the gate's baselines were scored with it. Nothing stops it.

**Cause.** There's no confinement (#10). A worker can edit any file in its worktree, including graders, goldens and baselines.

**Fix.** A PreToolUse fence hook in `<city-dir>/.claude/settings.json` that blocks writes to the goldens, the baselines and the fixture manifest. I left the product's prompts and eval code editable, because the spec required eval-code changes, and logged every edit to them with its diff. Read the fence back in `.gc/settings.json` after the first session starts, since the merge lands at start.

**How I verified.** Observed: the edit happened at milestone 1, before the fence. With the fence in place from milestone 2 on, a watcher found no write to a fenced path in any later run.

**Class.** Configuration.

## 37. A stock alert reports a normal build's bead growth as an escalation

**Symptom.** The human mailbox receives "ESCALATION: JSONL spike detected [HIGH]", with a bead-store growth of 27% to 43% against a 20% threshold. Nothing is stalled or waiting.

**Cause.** A stock city order alerts on fast bead-store growth, and a build creates beads fast.

**Fix.** Expect it. Record the mailbox's contents before the run so new messages stand out, and batch these alerts instead of treating each as a stall.

**How I verified.** Observed: two at milestone 1, one at milestone 2 and one during a setup smoke test.

**Class.** Configuration (a noisy default).

---

## Observed, not yet explained

- `ready_delay_ms` resolves to `10000` from `builtin:claude` even though `city.toml` sets `0`. `gc config explain --provider claude` shows it.
- Right after `gc init`, `gc status` reported `bead cache unavailable` and showed 2 of 4 agents as `unknown`.

---

## Sources

These are the private notes behind this page. "Setup log" is the experiment's setup record. "Incident review" is the write-up of the failed first launch. "Runbook" is the step-by-step setup procedure the log follows. "Research" refers to two read-only passes over the `gc` 1.5.0 docs, source and pinned pack. "Run log" is the Gas City driver's minute-by-minute record of the build, with milestone 1 on 2026-10-08 and milestone 2 on 2026-10-09. "Comparison record" is the experiment's shared log of both sides. "Misroute trace" (2026-10-10) and "check-scripts research" (2026-10-08) are read-only investigations of the `gc` 1.5.0 source and pinned pack. "Milestone-1 write-up" is the measured comparison behind [two-driver-results.md](../experiments/two-driver-results.md). For entries 27–37, "run C logs" are the reconfigured run's driver records for milestone 1 (2026-10-11), milestone 2 (2026-10-12), the recovery (2026-10-12 to 13) and the third attempt (2026-10-14). "Rulings" are an advisor session's filed decisions during that run. "Teardown log" is the record of stopping Gas City on 2026-10-15.

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
| 9 | Detached worktrees from `--detach HEAD` | Setup log §5.6; orchestrator research §C3–C5; run log, milestones 1 and 2 (item landings tables) |
| 10 | Guard hook is a shallow blocklist | Setup log §5.8 and §5.10 (hook read-back); orchestrator research §C1–C2 |
| 11 | Local pack clone isn't what runs | Orchestrator research §0; setup log §5.6 (pin sha) |
| 12 | `.gc/` and `worktrees/` not ignored | Orchestrator research §D2–D3 (code-read); comparison record, smoke B (2026-10-08 ~17:02Z) and the fix that followed; run log, milestone 1 drain close (13:30 MST); comparison record, milestone-2 rulings (~18:30Z, `check-ignore`) |
| 13 | `gc rig add` commits, then dirties `.beads/config.yaml` | Setup log, 2026-10-08 03:20Z and 03:26Z entries; runbook §5.4 and §5.10 |
| 14 | Context-bundle paths resolve from the bundle's directory | Setup log, 2026-10-08 03:22Z and 03:26Z entries; runbook §5.4; pinned pack's `validate_context_bundle.py` |
| 15 | `--include` of the roles pack displaces the default import | Setup log, 2026-10-08 03:42Z and 15:57Z entries; setup log §5.10 (duplicate import); runbook §5.10 |
| 16 | Sling target needs its rig prefix; dry run hides vars | Setup log, 2026-10-08 15:57Z entry; runbook §5.11 and its values table |
| 17 | Workflow roots routed to implementation workers | Misroute trace, §1–§4 (file:line at `v1.5.0`, upstream #6461); run log, milestone 2 (12:53, 13:40 and 14:13 MST stalls) |
| 18 | Finalize leaves work detached; publish can't deliver | Comparison record, finalize smoke (17:32Z) and milestone-2 rulings (~18:18Z prep publish, fast-forward); run log, milestone 1 item landings and milestone 2 publish close (17:58 MST); milestone-1 write-up, Delivery |
| 19 | Review loop can't converge on decisions it can't make | Run log, milestone 1 review (13:44–13:57 MST) and milestone 2 review iterations 1–6 (17:19–17:51 MST); comparison record, milestone-2 rulings (what Gas City routed to a human; delegation gap) |
| 20 | Agents ask a human in autonomous mode | Run log, milestone 1 (~15:58 MST prompt, answered ~17:09) and milestone 2 (12:53, 13:40, 14:13 MST); comparison record (no notification); milestone-1 write-up, attended time (52-minute block) |
| 21 | Artifact checks fail on PyYAML | Run log, milestone 1 (11:11 MST finding; interpreter check) and milestone 2 item landings (summary commit per retry); check-scripts research, bottom line and §(b) (#4239, packs#340) |
| 22 | Idle worker slots wake every ~50 s | Run log, milestone 1 (12:03–12:16 MST); milestone-1 write-up, Cost (idle drain $34.95, 165 sessions, 449 calls); misroute trace §2 |
| 23 | Mayor keeps the settings it started with | Comparison record, smoke B failure (2) and the fix that followed (`ps -E`) |
| 24 | Dispatcher restarts; worker session ids change | Run log, milestone 1 (11:28 and 11:36 MST restarts) and milestone 2 (14:00–14:15 MST session id change, false stall) |
| 25 | `claude plugin eval` sandbox stops the CLI | Run log, milestone 2 (14:00 MST worker report; gate checkpoint, layer 3); comparison record, milestone 2 (other driver's evals 8/8) |
| 26 | Shared usage limit pauses every agent | Run log, milestone 1 (stall #1, 14:06–15:34 MST; Mac sleep from `pmset`); comparison record (88-minute exclusion) |
| 27 | Claim race survives the #6461 workaround | Run C logs, every phase (stall tables); comparison record, milestone 1 (claim timing, 3–25 s); a read-only source read of workflow-finalize |
| 28 | Execution backstop kills active work | Run C logs, milestone 1 (two kills, pane-activity samples); a read-only read of the `gc` 1.5.0 execution backstop source; rulings on the stall; comparison record (threshold correction, milestone-2 batch with no kill) |
| 29 | Host sleep cuts an API call mid-turn | Run C logs, milestone 2 and recovery (stall, `pmset -g log` waits); comparison record (AC power profile) |
| 30 | Untracked final report collides at integrate | Run C logs, recovery (integrate close reason, finalize note); rulings on the integrate failure; run C logs, milestone 1 (untracked stage artifacts) |
| 31 | No per-step retry; reopen the bead | Comparison record, recovery delivery retry (re-sling rejected, reopen, claim timing); run C logs, recovery |
| 32 | Usage figure undercounts | Run C logs, all four runs (spend samples, sink copies per window); the run's spend-cap method, written before the run |
| 33 | Advisor rewrote history below the target tip | Run C logs, milestone 1 (advisor rulings, rewrite, integrate close reason); comparison record (`merge-tree` forecast, `bd` config restore); rulings after milestone 1 |
| 34 | `gc stop` unregisters; Dolt restarts | Teardown log; comparison record, 2026-10-15 |
| 35 | Supervisor is a launchd job | Teardown log; run C logs, third attempt (overnight event count) |
| 36 | Agents can edit what grades them | Run C logs, milestone 1 (prompt edit) and later runs (fence watch); rulings on the fence |
| 37 | Stock growth alert | Run C logs, milestones 1 and 2 (mailbox tables) |
| — | `ready_delay_ms`, `bead cache unavailable` | Setup log §5.6 and §5.7 |

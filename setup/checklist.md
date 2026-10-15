# Pre-flight checklist

Run this before the first `gc init` on a machine, and again before any run you plan to measure. Each line points to the gotcha it prevents in [gotchas.md](gotchas.md). Written against `gc` 1.5.0, and updated after the reconfigured run on 2026-10-15.

## Before `gc init`

- [ ] If a Claude Code session will run `gc` for you, switch it to default permission mode. Auto mode refuses the writes that configure Gas City's agents. (#1)
- [ ] Type any slash command that boots a session yourself. Don't relay or paste it. Use a fresh window. (#2)
- [ ] `dolt config --global --list` shows `user.name` and `user.email`. (#3)
- [ ] `git config --global beads.role` prints `maintainer`. (#8)
- [ ] `which claude` in the shell that will run `gc init` resolves to the real binary, not a terminal multiplexer's shim. If it doesn't, prefix `gc init` with an explicit `env PATH=...`. (#6)
- [ ] Credentials the agents need are in `~/.gc/secrets.env`, not only in your interactive shell. (#6)

## Right after `gc init`

- [ ] The output reaches `[8/8]`, and `gc cities` lists the city. (#3)
- [ ] Read the `PATH` in `~/Library/LaunchAgents/com.gascity.supervisor.plist`. The real `claude` comes first, and there's no shim directory and no API key in it. (#6)
- [ ] `city.toml` has a `[providers.claude]` block with `option_defaults` for `model`, `effort` and `permission_mode`. (#4)
- [ ] `gc config explain --provider claude` shows those values coming from `providers.claude`, not `builtin:claude`. (#4, #5)
- [ ] Note which pack sha the city pinned, and read that version of the pack, not your local clone. (#11)
- [ ] Restart the `mayor` session after the provider block is in place, and check its environment with `ps -E`. It started before your provider settings existed. (#23)

## Before the first `gc sling`

- [ ] Add the rig with no `--include`, then check that `gc config explain` lists `<rig>/gc.*` roles. (#15)
- [ ] Find `gc`'s own "bd init" commit on the rig's branch and count agent output from after your setup commits. `.beads/config.yaml` is `--skip-worktree`, and `git status` in the rig is clean. (#13)
- [ ] Allow time for `gc import install`. Mine took 23 minutes. (#7)
- [ ] `gc doctor` has no failures, and you've read every warning. (#8)
- [ ] You've planned your own integration step: find the item worktree that holds every item's commits, fast-forward the branch to it, and push. Turning on `push` and `open_pr` doesn't do this. (#9, #18)
- [ ] `/.gc/` and `/worktrees/` are in the repository's `.git/info/exclude`. (#12)
- [ ] The pack's check scripts are in `<rig>/.gc/scripts/`, and every `python3` the controller might resolve can import PyYAML. One trivial bead's artifact control closes `pass`. (#21)
- [ ] If you pinned a pack new enough to ship the check scripts, bash 4 or newer leads the check PATH. On a Mac, `/bin/bash` is 3.2 ([what-the-docs-say.md](what-the-docs-say.md#checks)).
- [ ] The goal bead says what the agents may spend without asking and who rules on questions about the bar. Otherwise the review loop iterates until it fails. (#19)
- [ ] Any context bundle passes the pack's validator, with item paths written from the bundle's own directory. (#14)
- [ ] The sling target is written `<rig>/gc.run-operator`, and every `--var` name matches a var declared in `gc formula show <formula> --json`. (#16)
- [ ] Any guard hook sits in `<city-dir>/.claude/settings.json` and still reads back after `gc rig add` and `gc import install`. Its limits are written down, because it won't stop an agent determined to write. (#10)
- [ ] Rehearse one trivial bead on a scratch rig first. Check the agent's actual command line, model and effort before you sling the real work.
- [ ] If you'll compare plugin evals, check that the cases can run inside `claude plugin eval`'s sandbox. A case that shells out to `claude -p` can't. (#25)
- [ ] The artifact root is clear in the rig checkout: earlier runs' records are copied out, and nothing untracked sits where the final report or an item will write. (#30)
- [ ] Every live step prompt tells agents never to end a turn while a background job runs. Otherwise the execution backstop kills long batches and gate runs. (#28)
- [ ] The host stays awake for the whole run. `pmset -g` shows no sleep on AC or battery, or `caffeinate -dimsu` is running, and the lid stays open. Plugging in isn't enough. (#29)
- [ ] The spend cap is priced from transcripts, by a collector of your own that calls `gc suspend`. Gas City's own usage figure reads low and late. (#32)
- [ ] A fence hook blocks agent writes to the goldens, baselines and anything else the gate reads, and it reads back in `.gc/settings.json` after the first session starts. (#36)
- [ ] Any advisor or fix step is told never to rewrite commits reachable from the target branch. (#33)
- [ ] You've recorded what's in the human mailbox, so new messages stand out. (#37)

## During the run

- [ ] Watch `gc events --follow` for `execution.step_stalled`, and `gc session peek` each stalled session. An agent may be waiting on a question nobody else will see. (#20)
- [ ] Expect workers to claim and release workflow roots, and idle slots to wake about every 50 seconds. Both cost money and neither needs a fix from you. (#17, #22)
- [ ] Key any stall watcher to the `gc` session, not a transcript file, and don't chase control-dispatcher restarts unless work stops. (#24)
- [ ] Keep the machine awake, and budget for a shared usage-limit pause. Take both out of any clock you compare. (#26)
- [ ] Expect claim races in every phase, even with the #6461 workaround. Leave the root alone; the item's finalize closes it. (#27)
- [ ] A `step_stalled` on a worker that's waiting on a background job means a kill within seconds, unless your prompts prevent it. (#28)
- [ ] After any sleep, look for agents whose pane says "went to sleep mid-response". The backstop will redo the step; count the redo. (#29)
- [ ] Read suspension from `.gc/runtime/suspension-state.json`, not `gc status`, which lags by seconds ([what-the-docs-say.md](what-the-docs-say.md#watching-a-run)).
- [ ] If integrate or publish fails for a reason you can fix, fix it and reopen that step's bead instead of re-slinging. Label the delivery human-assisted. (#31)

## Stopping

- [ ] Closing your Claude Code sessions stops nothing. The supervisor keeps every registered city running. (#35)
- [ ] Write down each city's path, then run `gc stop` on each one and `gc supervisor stop --wait`. (#34, #35)
- [ ] Check for leftover Dolt processes and stop them, and run no `bd` command afterward. (#34)

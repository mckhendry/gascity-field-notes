# Pre-flight checklist

Run this before the first `gc init` on a machine, and again before any run you plan to measure. Each line points to the gotcha it prevents in [gotchas.md](gotchas.md). Written against `gc` 1.5.0.

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
- [ ] The goal bead says what the agents may spend without asking and who rules on questions about the bar. Otherwise the review loop iterates until it fails. (#19)
- [ ] Any context bundle passes the pack's validator, with item paths written from the bundle's own directory. (#14)
- [ ] The sling target is written `<rig>/gc.run-operator`, and every `--var` name matches a var declared in `gc formula show <formula> --json`. (#16)
- [ ] Any guard hook sits in `<city-dir>/.claude/settings.json` and still reads back after `gc rig add` and `gc import install`. Its limits are written down, because it won't stop an agent determined to write. (#10)
- [ ] Rehearse one trivial bead on a scratch rig first. Check the agent's actual command line, model and effort before you sling the real work.
- [ ] If you'll compare plugin evals, check that the cases can run inside `claude plugin eval`'s sandbox. A case that shells out to `claude -p` can't. (#25)

## During the run

- [ ] Watch `gc events --follow` for `execution.step_stalled`, and `gc session peek` each stalled session. An agent may be waiting on a question nobody else will see. (#20)
- [ ] Expect workers to claim and release workflow roots, and idle slots to wake about every 50 seconds. Both cost money and neither needs a fix from you. (#17, #22)
- [ ] Key any stall watcher to the `gc` session, not a transcript file, and don't chase control-dispatcher restarts unless work stops. (#24)
- [ ] Keep the machine awake, and budget for a shared usage-limit pause. Take both out of any clock you compare. (#26)

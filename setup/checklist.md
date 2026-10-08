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

## Before the first `gc sling`

- [ ] Add the rig with no `--include`, then check that `gc config explain` lists `<rig>/gc.*` roles. (#15)
- [ ] Find `gc`'s own "bd init" commit on the rig's branch and count agent output from after your setup commits. `.beads/config.yaml` is `--skip-worktree`, and `git status` in the rig is clean. (#13)
- [ ] Allow time for `gc import install`. Mine took 23 minutes. (#7)
- [ ] `gc doctor` has no failures, and you've read every warning. (#8)
- [ ] You've decided how commits in `<rig>/worktrees/<id>` will get back to your branch. (#9)
- [ ] `worktrees/` is in the rig's `.gitignore` or `.git/info/exclude`. (#12)
- [ ] Any context bundle passes the pack's validator, with item paths written from the bundle's own directory. (#14)
- [ ] The sling target is written `<rig>/gc.run-operator`, and every `--var` name matches a var declared in `gc formula show <formula> --json`. (#16)
- [ ] Any guard hook sits in `<city-dir>/.claude/settings.json` and still reads back after `gc rig add` and `gc import install`. Its limits are written down, because it won't stop an agent determined to write. (#10)
- [ ] Rehearse one trivial bead on a scratch rig first. Check the agent's actual command line, model and effort before you sling the real work.

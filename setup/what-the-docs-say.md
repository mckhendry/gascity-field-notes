# What the docs say: configuring Gas City by the book

After two milestones went wrong in the ways [gotchas.md](gotchas.md) records, I stopped guessing and checked each problem against Gas City's own documentation. Four read-only passes covered the published docs, the docs source at the release tag, `gc <cmd> --help`, the pinned `gascity` pack, and, where the docs were silent, the `gc` source. This page is what they found: for each problem, what the docs actually say, whether configuration can fix it, and where you need code.

- **Versions:** `gc` 1.5.0 (tag `v1.5.0`), `bd` 1.3.1. The `gascity` pack pinned by 1.5.0 is `sha:3b3b89f2…`, called **the pin** below. Pack main is `520e95cb` (2026-10-08). Upstream `gc` main is `8fd6aa25c` (2026-10-10).
- **Date:** 2026-10-10.
- **Tags:**
  - **Documented**: the gc docs, CLI help, or the pack's own README/REQUIREMENTS say it.
  - **Code-read**: only the source or a pack prompt shows it.
  - **Contradicted**: the docs and the code or my runs disagree.
  - **Undocumented**: nothing authoritative says anything.
  - **Observed**: I also saw it happen on a live city.

Nobody ran a state-changing `gc` command for this page. Treat every Code-read item as a lead to verify on your own city.

---

## The short version

1. **docs.gascity.com documents upstream main, not the release you installed.** It serves pages that exist only on main. For 1.5.0, read `git show v1.5.0:docs/…` and `gc <cmd> --help`.
2. **`build-basic` never lands work on a branch, and no setting changes that.** It's an unfinished design, not a misconfiguration. See [Getting work onto a branch](#getting-work-onto-a-branch).
3. **`interaction_mode=autonomous` is allowed to stop and ask.** The pack's decide-or-fail mode is `headless`.
4. **Two of my fixes need code:** a spend cap (none exists) and the workflow-root routing bug that causes both the misrouting and the idle churn.
5. **Everything else is configuration:** a newer pack pin, a host Python fix, per-rig patches, formula and prompt overrides, orders, and keeping the Mac awake. Several of those are undocumented.

| Problem (gotcha #) | Verdict | Fix |
|---|---|---|
| Work never reaches the branch (9, 18) | Config, but effectively a pack fork | Override worktree prep, add an integration step, push the recorded commit. Or use the gastown pack's refinery path |
| Agents stop to ask a human (20) | Config, partly | `interaction_mode=headless`, plus shadowed stage prompts, plus a harness-level block on asking |
| No one is told about escalations (20) | Config | An order on `execution.step_stalled`, plus an escalate-label pattern |
| Workflow roots routed to workers (17) | **Code** (upstream #6461, #6932) | An interim order that strips the route from expanded roots |
| Idle wake every ~50 s (22) | **Code**, same root cause | Cap pools and lower effort to shrink the blast radius |
| Unbounded pools, effort `max` (4) | Config | `[[rigs.patches]]` with `max_active_sessions` and `option_defaults` |
| Per-run spend cap | **Code**: none exists | A usage-sink script that suspends at a threshold |
| `gc costs` shows $0 | Config | Add `[[pricing]]` for your model |
| Review loop gives up at 6 (19) | Config | Formula override. The 6 is hardcoded, and `max_iterations` isn't wired |
| An advisor that rules on review questions | Config | A new step routed to it, in an overridden review formula |
| Missing check script (21) | Config | Pin the pack at or after `d36950e` |
| PyYAML invisible to checks (21) | Config (host) | Install PyYAML into the check interpreter's main site-packages |
| A failed check still releases its dependents | **Code** (open PR #7174) | Fix the false failures instead |
| Guard hook is only a blocklist (10) | Config | A sandboxing runtime backend, or a tighter permission mode plus deny rules |
| Mac sleep pauses everything | Config (host) | `caffeinate` or `pmset` |

---

## Which docs to read

- **The published site tracks main** (Documented). It serves `/getting-started/upgrading`, which exists only on main, and Tutorial 01 shows the `[defaults.rig.imports.gc]` block that the 1.5.0 docs leave out. That block is also what the 1.5.0 binary actually writes, so in this case the 1.5.0 docs are stale.
- **One documented 1.5.0 step is wrong** (Contradicted). After editing `~/.gc/secrets.env`, the 1.5.0 docs say to run `gc service restart`. In 1.5.0 that command restarts a *workspace service by name*, not the supervisor. Main corrects it to `gc supervisor install`, which rewrites the service file and restarts it on change.

## Before the first sling

- **Prerequisites the docs never state** (Undocumented):
  - A global Dolt identity. `gc init` stops at 6 of 8 without one (gotcha 3; the error text is the only documentation).
  - `git config beads.role maintainer`. It appears only as sample output in Tutorial 03 (gotcha 8).
  - Python with PyYAML, for the pack's checks.
  - A minimum Claude Code version. None is given.
- **Built-in Claude defaults** (Code-read): `permission_mode = "unrestricted"` (`--dangerously-skip-permissions`) and `effort = "max"`. The options schema lists `auto-edit` as the default, but the provider's `option_defaults` override it. Set both yourself (gotcha 4).
- **`gc init --no-start`** (Documented, but only in CLI help) writes the city without registering or starting it. Use it to put your provider block in `city.toml` before any agent launches, the always-on mayor included. That avoids gotcha 23, which no guide connects to this flag.
- **`ready_delay_ms = 0` is a no-op** (Code-read). The field is a plain int, so 0 reads as unset, and the built-in 10000 wins. The template still writes 0.
- **There is no pre-sling checklist in the docs.** The quickstart is init, `rig add`, sling. [checklist.md](checklist.md) is mine.

## Rig hygiene

- **`gc rig add` ignores only `.beads/*` and `!.beads/identity.toml`** in the rig (Code-read; `cmd/gc/gitignore.go`). Nothing ignores the rig's `.gc/` or `worktrees/`, and no doc mentions gitignore or `.git/info/exclude` at all (gotcha 12). Add `/.gc/` and `/worktrees/` to `.git/info/exclude` yourself.
- **`bd init` commits to the rig's checked-out branch** (Undocumented; gotcha 13). `bd` has `--stealth` and `--setup-exclude`, but `gc` doesn't pass them.
- **`--include` replaces the default rig imports rather than adding to them**, and defaults are copied in only at a fresh `rig add` (Code-read; gotcha 15). The docs imply add-time-only but never say `--include` suppresses them.

## Pack sourcing and overrides

- **Import shapes** (Documented): a local path relative to the importing pack, or a remote source with an optional semver constraint or exact `sha:` pin. Prefer `/tree/<ref>/<path>` URLs for GitHub subpaths.
- **`gc import add` on a path inside a git checkout locks to that checkout's HEAD** (Documented in CLI help). Uncommitted edits are invisible. For a live local clone, write the path into the TOML by hand. A path outside any git repo is stored as a plain path.
- **The built-in pin is gascity pack 0.1.6, from 2026-06-16**, served offline from the binary (Documented). Your local clone is not what runs (gotcha 11). A 23-minute `gc import install` on all-bundled pins contradicts the "offline" claim, and the cause is unexplained (gotcha 7).
- **Three documented ways to customize without forking:**
  - `[[patches.agent]]` or `[[rigs.patches]]` for agent fields.
  - Formula layering: the importer beats its imports, and `[[rigs]] formulas_dir` beats everything. Check which layer won with `gc formula show <name>`.
  - Shadowing a step's prose by placing a file at the same `assets/workflows/<formula>/<step>.md` path in a higher layer. The pack's README lists the override points it treats as stable.
- **`gc formula catalog` exists in 1.5.0 but is hidden** (Code-read). It's missing from `gc formula --help`, even though the pack README tells you to run it. Nothing documents which pack versions work with which `gc`.

## Getting work onto a branch

This is why my Gas City runs delivered nothing (gotchas 9 and 18).

- **No drain, formula or launch setting is about git** (Documented). The drain keys are `formula`, `context`, `member_access`, `max_units`, `on_item_failure`, `continuation_group` and `item.single_lane`. The launch variables (`push`, `open_pr`, `drain_policy`, …) don't name a branch or merge target either.
- **Core never creates worktrees. An agent does, from a pack prompt** (Code-read). The `do-work` step `prepare-worktree` always makes a detached checkout. In the pin it's `--detach HEAD`. On pack main it's `--detach origin/<origin's default branch>`, which bases a feature-branch run on main instead.
- **Nothing in `build-basic`, `do-work`, `implement` or `publish` merges anything** (Code-read). The finalize and review-fix prompts treat the approved worktree as the deliverable and say publish "owns propagation". Publish is three short prompts that never say which commit to push.
- **This is an unfinished design, and upstream says so** (Documented as open work):
  - The archived worktree roadmap has "merge-back flow" unchecked.
  - RFC gascity-packs#209, "Deterministic Worktree Lifecycle", names the gap and proposes an integration step plus a rule that publish pushes "exact recorded commits, never ambient HEAD". Its implementation, PR #214, is unmerged.
  - Open pack PRs #421 and #422 propose a different workspace model.
- **Shipped paths that do land work** (Documented):
  - The gastown pack's refinery merges a polecat's feature branch (`merge_strategy` direct or mr).
  - Core's `mol-polecat-commit` pushes `HEAD:base_branch`.
  - The docs map Gas Town's Refinery to "a workflow step, not a standing role".
  - `gc sling --merge` is read only by gastown; the `gascity` pack ignores it.
- **`gc worktree ensure` already ships in 1.5.0** (Documented). It creates an attached per-bead branch from `--base` and never a detached one, but no pack calls it.
- **`drain_policy=same-session` isn't a workaround on 1.5.0** (Contradicted). The docs promise one shared worktree, but nothing creates it (gascity#5429; fix in open pack PR #347).
- **What "approved" means for publish** (Documented): the final report's `status: approved`. Downstream stages consume only approved artifacts. "Nothing approved to push or PR" was the publisher agent's own wording after finalize recorded `blocked`.

**Fix by configuration:** in a higher-priority local pack layer:
1. Shadow `prepare-worktree` to create an attached branch from your target (`gc worktree ensure --base <target>` or `git worktree add -b`).
2. Add an integrate step to an overridden `build-basic` that merges the closed items' branches in dependency order and records the commit.
3. Shadow publish's push to push that recorded commit.

No `gc` code is needed, but it's a fork in all but name.

**From scratch:** skip `build-basic`'s drain for delivery and use the gastown polecat and refinery path. You lose `build-basic`'s plan, review and finalize scaffolding, or rebuild it as plain slung beads.

## Autonomy, escalation and an advisor

- **`interaction_mode` is a pack variable; `gc` never reads it** (Code-read).
- **The pack defines the modes** (Documented in pack REQUIREMENTS):
  - `autonomous` may still block "for unsafe actions, missing required evidence, or external-state requirements". A missing PyYAML and a "paid rerun needs approval" both qualify.
  - `headless` never asks and fails closed with machine-readable reasons.
- **The pack doesn't honor its own rule** (Contradicted). Its requirements say every gate must honor the mode, but in `build-basic` only the requirements prompt reads it. Review, implement, do-work and finalize prompts never see it.
- **Nothing in `gc` stops a Claude session from asking a question** (Code-read). Human escalation exists only for maintenance scripts. It goes to a `human` mailbox that wakes nobody (Documented). No Gas Town-style Overseer inbox ships.
- **Notification is buildable** (Documented: orders):
  - An order triggered by the `execution.step_stalled` event can run a notify script. That event exists in code but isn't in the docs' event list, and it fires about 11 minutes into a stall.
  - The `oversight-rig` pack shows an agent-raised version: the agent labels a bead `severity:escalate`, and a condition order delivers it.
- **An advisor agent alone does nothing** (Documented by absence). Agents coordinate only through beads and mail, and nothing lets one agent ask another and wait. An advisor that rules on review questions works only as a new graph step routed to it, inside an overridden review formula. Check that formula is on the pack's stable override list first.
- **Pre-authorizing paid reruns** belongs in the requirements, the context bundle, or shadowed review prompts. `gc` has no approval primitive. For a real human gate, use `[steps.gate]`/`waits_for`, which makes the wait explicit instead of an agent stalling in its terminal.

**Fix by configuration:**
1. Launch with `interaction_mode=headless`.
2. Shadow the stage prompts that ignore the mode, with a rule like "never ask; on a failed last attempt record `gc.outcome=fail` with a reason and close."
3. As a backstop, block the ask tool at the harness (`[providers.claude] args`). That's a Claude Code flag, not a `gc` knob, so check how it combines with any wrapper command.

## Pools, effort and cost

- **The pack's 14 roles are unbounded** (Code-read). Each sets only `description`, `scope = "rig"` and `fallback = true`. `fallback` was removed from `gc` and is silently ignored (Documented). None sets a pool cap, model or effort, so each is an unlimited pool at effort `max`.
- **Patch them per rig** (Documented): `[[rigs.patches]]` with `max_active_sessions` and `option_defaults = { model, effort }`.
- **The misrouting and the ~50-second idle churn are one bug** (Code-read; upstream #6461, with #6932 for the outer root; both open on main). The spec says a workflow root is controller-owned and blocked until finalize. The code stamps every graph root `gc.routed_to` and doesn't block it. So each drain item's root is claimable by the worker pool, and counts as pool demand, from the moment it's created (gotchas 17 and 22). An upstream comment reproduces it on 1.5.0 with this pack and formula.
  - **Interim workaround** (from the issue thread, not the docs): an order that unsets `gc.routed_to` on expanded workflow roots.
  - Pool caps and lower effort bound the cost but don't stop the churn. No interval setting addresses it.
- **No spend budget exists** (Undocumented; checked on 1.5.0 and main). There is no config field, flag or order setting for one; `gc`'s budgets are time, wake and attempt budgets. The nearest workaround: `[usage] provider = "exec:<script>"`, with a script that sums cost per run and calls `gc suspend` at a threshold. That stops writing `.gc/usage.jsonl`, so `gc costs` goes blank unless the script also writes it.
- **`gc costs` shows $0 for newer models** (Documented and Code-read). Built-in prices stop at `claude-opus-4-8`, and unpriced calls are left out of the total. Add a `[[pricing]]` entry. Treat the result as a floor: the usage design doc lists ways it undercounts against transcripts.
- **The review loop's 6** (Code-read): `build-basic-review` hardcodes `max_attempts = 6` and never reads the `max_iterations` variable (gotcha 19).

## Checks

- **The missing check script is already fixed in the pack** (Code-read; gotcha 21). The pin declares `.gc/scripts/checks/build-artifact-valid.sh` with no backing file (gascity#4239, still open). Pack commit `d36950e` (2026-10-04) switched every formula to `../assets/scripts/checks/...`, which `gc` 1.5.0 already resolves across formula layers. The docs describe that resolution only for `description_file`. Pin the pack at or after `d36950e` instead of hand-installing scripts. That brings about 20 other pack commits with it. The published formulas guide still shows the broken path.
- **Checks run in a stripped environment** (Code-read; not in the docs):
  - `PATH` is the directories holding `bd`, `gc`, `dolt` and `jq`, then `/usr/local/bin:/usr/bin:/bin`.
  - `HOME` is the city directory.
  - No config key adds variables or PATH entries.
- **That's the PyYAML failure** (Observed and reproduced). On my Mac, `python3` on that PATH is `/usr/local/bin/python3`, whose PyYAML was a `--user` install. Python finds user installs through `HOME`, which now points at the city, so `import yaml` fails.
  - **Fix:** install PyYAML into that interpreter's main site-packages.
  - **Verify:** `env -i PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin HOME=/tmp python3 -c 'import yaml'`.
  - The install docs never mention Python. That gap is tracked in gascity-packs#340, still open.
- **A failed check still releases its dependents** (Code-read; Contradicted by the published guide, which says exhaustion "blocks downstream work"). Downstream steps run anyway, then `workflow-finalize` fails the whole run. The fix-closed change is PR #7174, still open. Retry counts come from the formula's `max_attempts`; change them by shadowing the formula. Exit code 75 re-runs without spending an attempt (Documented).

## Safety

- **"A feature, not a sandbox"** (Documented). Gas City treats config, packs and providers as trusted code.
- **Confinement comes from a different runtime backend** (Documented): `exec:` sandbox packs, `k8s` or `ssh:`. The default `tmux` runtime runs agents as you.
- **Settings precedence** (Code-read): your `<city>/.claude/settings.json` is merged into the managed `.gc/settings.json`. That's where deny rules and hooks go.
- **No recommended permission settings** (Undocumented). The docs don't recommend any Claude Code permission or sandbox settings. A tighter mode than `unrestricted` (`auto-edit`, or `full-auto`, which maps to `dontAsk`) plus deny rules is untested for unattended runs.

## Watching a run

- `gc events --follow` and `--type`, for example `--type execution.step_stalled` (Documented flags; that event type isn't listed in the docs).
- **Check failures:** `gc.attempt_log` on the step's control bead, and `.gc/runtime/control-dispatcher-trace.log` (Code-read). `gc trace` covers only the session reconciler, not workflow steps.
- **Following an agent:** `gc session logs <session> -f`, `peek` or `attach` (Documented). `-f` picks its transcript file once at start, so re-run it after a worker changes Claude session id (Code-read, untested; gotcha 24).

## Long unattended runs

- **Sleep** (Undocumented). Nothing covers host sleep, and `gc` takes no power assertion. The launchd service only restarts the supervisor after a crash. Keep the host awake with `caffeinate -dimsu` or `pmset`.
- **Usage limits** (Code-read). A usage-limit dialog at session start is dismissed, and the session is marked `sleep_reason=rate_limit`. A limit hit mid-turn is left to Claude Code (gotcha 26).
- **Restarts resume** with `--resume <session-id>`, and step state lives in beads (Code-read and Documented).

## Teardown and a second city

- **`gc stop` also unregisters the city** (Documented). It stops agents and Dolt too. `gc unregister` leaves sessions running. `gc rig remove` only edits `city.toml` and `site.toml`. `gc import prune` clears unreferenced pack cache.
- **No documented "delete and start fresh" procedure.** Nothing covers the city directory, a rig's `.beads/`, the `bd init` commit, a rig's `.gc/` and `worktrees/`, or orphan Dolt databases. Under the default proxied Dolt, any `bd` read restarts Dolt, so a stray `bd` command can bring a stopped city's database back.
- **Two cities can share a machine** (Documented): one supervisor, one API port, a tmux server per city. But `gc init` or `gc register` beside a running city can kill and respawn the supervisor, which cycles the other city's in-flight work (Code-read). Pass `--city` everywhere.

---

## Questions the docs don't answer

1. The minimum Claude Code version, and which CLI flags `gc` depends on.
2. Whether `build-basic` is meant to deliver, or only to produce an approved worktree for a human or a refinery to land. Its prompts disagree with each other.
3. Whether a drain passes parent variables (such as a target branch) into item formulas.
4. Whether `gc worktree ensure` is meant to be called from an agent prompt.
5. Which way the maintainers want to fix the worktree lifecycle: RFC #209 (and its unmerged PR #214), or the workspace model in PRs #421/#422.
6. Whether `drain_policy=same-session` avoids the routed-root churn.
7. Whether `full-auto` plus deny rules lets the pack's roles finish unattended.
8. Whether a provider-config change restarts an already-running mayor.
9. How to delete a city and its proxied Dolt data so it can be recreated at the same path.
10. Which `gascity` pack revisions are compatible with which `gc` releases.

## Sources

- **`gc` docs** at tag `v1.5.0` and on main: `docs/getting-started/{installation,troubleshooting,coming-from-gastown}.md`, `docs/tutorials/01-07`, `docs/guides/{understanding-formulas,understanding-packs,shareable-packs,configuring-an-agent}.md`, `docs/reference/{config,cli,trust-boundaries,events}.md`, `docs/reference/specs/{formula-spec-v2,pack-spec}.md`.
- **Engineering docs:** `engdocs/archive/backlogs/worktree-roadmap.md` and `engdocs/design/usage-facts-v0.md`.
- **The `gascity` pack** at the pin and at `33da48c`: `README.md`, `REQUIREMENTS.md`, `formulas/*.formula.toml`, `assets/workflows/**`, `roles/agents/*/agent.toml`.
- **Source files cited above:** `cmd/gc/gitignore.go`, `internal/worker/builtin/profiles.go`, `internal/convergence/condition.go`, `internal/formula/parser.go`, `internal/graphroute/graphroute.go`, `internal/hooks/hooks.go`.
- **Upstream issues and PRs:** gastownhall/gascity #4239, #5429, #6461, #6932, #7174; gastownhall/gascity-packs #209, #214, #340, #347, #421, #422.
